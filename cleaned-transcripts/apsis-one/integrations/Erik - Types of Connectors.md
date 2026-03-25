---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [Connector Architecture, Legacy Connectors, Generic Connector, Third-party Integrations, Integration Configuration, Outbound Flow]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Shreevidhya Ganesan]
key_components: [Justin (integration platform), Legacy Connectors, Generic Connector, Third-party Integrations, Installer Options, Field Mappings, Subscription Mappings, Outbound Mappings, Consent Synchronization]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session covers the three main types of connectors in the Apsis One integration platform (codenamed **Justin**). Erik Andersson, a senior architect, explains the evolution from **legacy connectors** (built in-house starting in 2018) to the **generic connector** approach (adopted around summer 2022), and discusses **third-party integrations**. The session emphasizes why the legacy approach became unsustainable and how the generic connector shifts responsibility to CRM developers to provide standardized endpoints. The discussion includes configuration details around installer options, field mappings, and outbound flows for syncing form submissions and consent changes to CRM systems.

---

## Background: Why We Have Multiple Connector Types

### Historical Context: The Legacy Connector Era (2018 onwards)

When the integration project began in late 2018, Apsis built connectors in-house by directly consuming existing CRM APIs. [Erik Andersson]:

> When we started this project back in 2018, I was a consultant here in late 2018 for this. We started building the connectors in house. So for example we wanted to connect to Microsoft Dynamics and the way we did this is that we utilized the existing APIs inside of Microsoft to extract whatever data we needed.

For each CRM integration, the Apsis team had to:
- Extract data using the CRM's native APIs
- Handle data transformation (e.g., converting 1/0 to true/false for booleans)
- Adapt to CRM-specific quirks and data inconsistencies
- Replicate new features across every connector individually

### The Maintenance Problem

This approach became unsustainable as more connectors were added:

> When you get up to like even 4 integrations then you will notice that this is not a sustainable way of developing this because like just adding one new feature would take like months of work and then you have the maintenance of them all will be a absolute nightmare.

**Current legacy customer base:** Approximately 60 customers total across legacy connectors:
- **Microsoft Dynamics:** substantial customer base
- **Efficy Enterprise 12.0:** large enterprise customer base (these are on-premise customers reluctant to upgrade)
- **Lime CRM:** 1-2 customers, not large but paying customers

**Policy:** No new legacy connectors are being built, and no new features are being added to existing legacy connectors. The only exception would be if a very large customer paid significantly to develop new features.

---

## Architecture: Three Types of Connectors

### Legacy Connectors (Microsoft Dynamics, Efficy Enterprise 12.0, Lime CRM)

**Characteristics:**
- Full-blown CRM integrations built and maintained by Apsis
- Support field mappings, contact/record retrieval, consent synchronization
- All data transformation and type conversion handled in Apsis code
- Single feature addition requires changes in every connector

**Supported by:** Microsoft Dynamics, Efficy Enterprise 12.0 (called FSC Enterprise in discussion), Lime CRM

**Key limitation:** Unsustainable maintenance model

---

### Generic Connector (Recommended Approach)

#### Design Philosophy

The generic connector inverts the responsibility model. Instead of Apsis adapting to each CRM, CRM developers must adapt to a **standardized Apsis contract**. [Erik Andersson]:

> The generic connector shifts the responsibility to the CRM instead. For the generic connector, we have like 1 connector in APSIS which is literally called generic connector. Where we Add all of the support for.

#### How It Works

Apsis defines a single standardized interface that all CRMs must implement:
- Consistent endpoint naming patterns (e.g., `/records/<page_number>`)
- Standardized response formats and data types
- Feature capability declarations (CRM tells Apsis what it supports)

When Apsis needs data, it calls the same endpoint for all CRMs; only the hostname differs.

```
GET https://<crm-hostname>/records/page=1
Expected response format (same for all CRMs):
{
  "records": [
    {
      "id": "<unique-id>",
      "email": "user@example.com",
      "firstName": "John",
      "created": "2025-01-15T10:00:00Z"
    }
  ]
}
```

#### Data Type Enforcement

Unlike legacy connectors, the CRM **must** send data in the correct types:

[Erik Andersson]:

> We force them to send us data in the format. They say like if it is supposed to be a boolean, then it has to be a boolean. If it is an integer, it has to be an integer. If it is a float, it has to be a float.

**No type coercion is performed in Apsis.** This prevents bugs like those encountered with Efficy Enterprise 12.0, which sent stringified integers and booleans instead of proper types.

#### Benefits

1. **Maintainability:** New features added once, automatically available to all CRMs
2. **Debugging:** Easy to isolate failures to a specific CRM implementation
3. **Scalability:** Adding a new CRM is straightforward
4. **CRM Ownership:** CRM developers understand their system better than Apsis

[Erik Andersson]:

> And by doing this now we can just add a new feature after new feature after feature because we add it in one place, we maintain like in one place for every CRM that have adapted to it and it is incredibly more stable and of course from a debugging perspective. It get it gets very easy to pinpoint the blame, so to say, because if you can see that the generic connector flow works for say tribe, but when it comes to E deal it is behaving in a weird way. Then it is easy to see, yeah, but then the problem exists in ED because it is working as it should in tribe or web CRM.

#### Timeline and Adoption

The generic connector project began around **summer 2022** (approximately 2 years before this session). [Shreevidhya Ganesan]:

> Basically summer 2022 we started this project I guess like you know of having the generic connectors and since then we have been developing only generic connector, right Eric?

**CRMs currently using generic connector:** Tribe, E-deal, Web CRM, Efficy Enterprise 12.1

**Migration status:**
- All new customers use generic connector
- Legacy 12.0 customers exist and cannot be forced to upgrade (on-premise installations)
- Efficy Enterprise 12.1 is the new standard version

#### Feature Capability Declaration

Each CRM instance declares what features it supports. When Apsis calls the list integrations endpoint, the CRM responds with:

```json
{
  "can_sync_email_activities": true,
  "can_sync_sms_activities": true,
  "can_sync_event_tool_activities": false,
  "supports_outbound_mappings": true
}
```

Apsis then uses this information to:
- Enable/disable UI options in the customer's section
- Control whether to send certain event types to the CRM

[Erik Andersson]:

> When you call integration and say like please list every integration that might be installed on this section. As part of this we are providing you with the information like can sync e-mail activity, can sync SMS activity, can sync Event tool activities. This information we retrieve from the CRM system like they tell us hello, I can sync e-mail, I can sync Event tool, I can sync this, I support this, yada yada yada and then this is propagated up to the service in APSIS so we can enable or disable disable that.

#### Versioning Approach

The generic connector does **not use traditional versioning**. Instead:
- Apsis adds new features and doesn't remove old ones
- CRMs declare support via feature flags
- Apsis only sends/expects data for features the CRM has declared support for

[Erik Andersson, responding to versioning question]:

> So we if we add something new in apsis that the CRM does not support, like we don't start sending that until they have enabled that feature flag in the CRM.

---

### Third-party Integrations (SuperOffice, Intermail, Join, LeadFamily, WooCommerce)

Third-party integrations are built and maintained by **external companies**, not Apsis. Apsis's role is minimal.

#### Apsis Responsibilities

When a third-party integration is installed, Apsis:
1. Creates a **keyspace** for the integration
2. Whitelists the CRM ID attribute for that keyspace
3. Generates a **single API key** that the third-party uses
4. Displays the API key in the UI so the customer can copy-paste it into the third-party's configuration

After these steps, **Apsis is done.** The third-party handles all logic, maintenance, and customer support.

[Erik Andersson]:

> We create the key space for them. So like if you install Super Office which is a third party one. We create the Super Office key space. We whitelist the CRM I oops. We whitelist the CRM ID attribute for this key space and we also generate A1 API key, but we don't. Like if they're if the sinks are not happening as they should, like we don't support any of that because we don't know how they how they work.

#### Current Customer Numbers (approximate)

- SuperOffice: 5-6 customers
- Intermail Loyalty: 1 customer (being discontinued)
- Join: 2-3 customers
- LeadFamily: small number
- WooCommerce: 3-4 customers
- **Total:** fewer than 10 customers across all third-party integrations

#### The Intermail Exception: Generic Connector Support

**Intermail** is an exception—it's a third-party that has implemented full **generic connector support**. When Intermail is installed:

1. Customer provides Apsis with Intermail's API key
2. Apsis uses this key to call Intermail's system (which supports the generic connector contract)
3. Apsis sets up webhooks and field mappings just like with native generic connectors
4. Intermail handles data transformation on their side

[Erik Andersson]:

> In their new join CX connector here they have added like a full generic connector support. So here it works like if you were to install tribe or dynamics or anything like you make the installation and then you can set up the field mapping and the consent mapping out.

Unlike other third-party integrations, Intermail's API is called by Apsis, not the other way around.

#### Why Third-party Integrations Exist

These are legacy remnants from when Jonas (a former product owner) attempted to partner with external companies. Modern strategy should avoid this model.

---

## Integration Configuration: Installer Options

Every CRM requires configuration specific to its data model and capabilities. This configuration is stored in **installer options** (or **connector options**).

### Key Configuration Parameters

#### Display Name and Integration ID

Each connector must have:
- **Display name:** What users see in the UI
- **Integration ID (logical ID):** Unique identifier used internally to differentiate from other connectors

#### Feature Toggles

For each supported activity type, Apsis can disable at the system level if a CRM doesn't support it:

- `can_sync_email_activities`
- `can_sync_sms_activities`
- `can_sync_event_tool_activities`

If disabled globally, Apsis will not even check the CRM instance to see if it supports the feature.

#### Concurrency and Performance Settings

- **Thread count:** How many parallel threads to use when downloading contacts/records
- **Page size:** How many records to fetch per API request

#### Profile Entity Configuration

Every CRM has a primary entity representing customers:
- **Microsoft Dynamics:** "Contact"
- **Tribe:** "Person"
- **Efficy Enterprise:** "Contact"

Configuration includes:
- Entity name
- Display name
- Whether to enable **inbound** sync (download from CRM to Apsis)
- Whether to enable **outbound** sync (send from Apsis to CRM)

#### ID Field Name (Critical)

The CRM must declare which field is the **unique identifier** for records. This field value is used as the Apsis **key** to identify and update profiles.

[Erik Andersson]:

> And an important part is also the ID field name, because this is the unique identifier. As you know in apps is like you always need to specify like a key space and a. Key and whatever value is contained here in the ID field name. This is the unique identifier like the key that we utilize in apps and identify the records with and for in the future if they send us updates, we will update the profile on that specific key and not create a duplicate duplicate with it.

Common values:
- "ID"
- "contact_id"
- "identifier"
- CRM-specific naming conventions

---

## Data Synchronization Directions

### Inbound (CRM → Apsis)

When **inbound is enabled**, Apsis requests records from the CRM and imports them as profiles into Apsis.

**Rule of thumb:** The CRM is the primary source of truth for **attributes** (contact details, company, etc.).

### Outbound (Apsis → CRM) - Special Rules

Outbound is NOT a simple mirror of inbound. There are specific flows where Apsis sends data to the CRM:

1. **Consent synchronization** (bidirectional - always happens if subscription mapping exists)
2. **Form submissions** (if outbound enabled for that entity)
3. **Event Tool activities** (if the CRM supports it)

#### Consent Synchronization (Critical Bidirectional Flow)

Consent must stay synchronized in both directions. [Erik Andersson]:

> Consents is a exception to this because consent must be the same in both the CRM and in APSIS. Usually like whenever you have an integration like the the rule of thumb is that the CRM is the like main source of the data, whatever, whatever is in the CRM. Should be what is in apsis, but this only holds true for the attributes because when it is, when as far as consent is.

**Scenario that demonstrates why bidirectional consent is critical:**

1. Customer installs Dynamics integration and imports 10,000 profiles
2. Customer sends an email campaign from Apsis
3. A recipient clicks the unsubscribe link in the email
4. Apsis marks the profile as opted-out
5. **If we don't sync this back to Dynamics:** Next sync from Dynamics would re-import the profile with opt-in status
6. **Result:** Profile is re-opted-in, and Apsis sends more emails, which violates the customer's explicit opt-out

[Erik Andersson]:

> What happens otherwise well. The profile in APSIS will be opted out if they press on unsubscribe, but the consent would still be opt in in the CRM system and the next time we do a sync to APSIS for whatever reason, then we would override their opt out with a opt in and now we would. Start sending things which they have explicitly said. You are not allowed to contact me on this and that would put this in a lot of trouble.

**Ways consent can change in Apsis:**
- Unsubscribe link in email
- Customer service representative (GDPR/manual opt-out)
- Task node in Marketing Automation workflow

All of these changes must be synced back to the CRM if a **subscription mapping** exists.

#### Attribute Changes: NOT Synchronized to CRM

[Erik Andersson]:

> We never sync attribute changes in apps to the CRM for this reason also because if you want to change attributes on the profile then you should change it in the CRM system because that is main source of the data.

This is by design: CRM is the source of truth for profile attributes. If you want to update an attribute, you do it in the CRM, and the next sync brings it to Apsis.

---

## Outbound Mappings and Form Submissions

### The Problem Outbound Mappings Solve

When a customer submits a form in Apsis, Apsis can send that submission event to the CRM to create a lead or opportunity. However, CRM systems have specific required fields:

**Example CRM "Lead" entity structure:**
- `first_name` (required)
- `last_name` (required)
- `email` (required)

**Example Apsis form submission:**
```json
{
  "eric_favorite_color": "blue",
  "my_super_cool_field": "John",
  "email_address": "john@example.com"
}
```

**Problem:** How does the CRM know that "my_super_cool_field" is actually the first name? Without explicit mapping, the CRM cannot populate the lead correctly.

### Solution: Outbound Mappings

Outbound mappings let the Apsis administrator declare the reverse of the field mapping:

**Field Mapping (inbound):** CRM field → Apsis attribute
**Outbound Mapping (outbound):** Apsis attribute → CRM field

When a form is submitted and outbound is enabled, Apsis:
1. Retrieves the profile's attributes from the form submission event
2. Maps them according to the outbound mapping configuration
3. Sends the mapped data to the CRM as a supplementary property on the event

[Erik Andersson]:

> The outbound mappings here lets you map essentially the reverse of the field mapping. So we take what field in apsis should be mapped to which field in the CRM system. So say that you create now create a form and you create a custom field. You map that custom field to a attribute in abscess as you can do. What we will try to do if you have set up the outbound mappings is that we will take the attributes on the profile that has the submit event. And then uh create like a a Jason object with um the data in this apps is field.

### Outbound Enabled: The Gate for Form Syncing

When you enable the **"Sync to CRM"** checkbox in a form's settings:

[Erik Andersson]:

> If I select this CRM sync, these outbound mappings they just regulate. Do I enhance this request to the CRM? We have like data according to this mapping or not like it will still work.

**If outbound is disabled:**
- Form submissions are sent to the CRM, but with minimal data
- The CRM receives only the raw form submission
- The CRM has to guess which form field maps to which lead field

**If outbound is enabled:**
- Form submissions include enriched data mapped according to outbound mappings
- CRM receives explicit field names and values
- CRM can populate lead records more accurately

### Event Tool Syncing

The **Event Tool** in Apsis can also trigger outbound sync to the CRM. A feature flag controls whether this option is visible in the Event Tool UI:

[Erik Andersson]:

> You have. Let's see. Sync Event Tool. It's events to events to CRM.

When an event is created in the Event Tool and the user selects **"Sync to CRM"**, Apsis:
1. Creates a listener in Audience (the event bus)
2. When the event fires, captures the event data
3. Sends it to the CRM system
4. CRM records the activity

**Note:** As of this session, there was discussion about whether an Event Tool feature flag on the account level is necessary, or if Apsis should rely entirely on the capability declaration from the CRM. [Erik Andersson recommends]:

> I I I don't know if you would call it versioning but. We we we don't start sending them, sending things to them before they have told us that now you can start sending it.

---

## Efficy Enterprise: Legacy vs. Generic

### Efficy Enterprise 12.0 (FSC Enterprise 12.0)

- **Legacy connector** (built by Apsis)
- Notorious for data inconsistencies:
  - Default date of **1899-12-31** for empty date fields (causes customers to appear 100+ years old)
  - Stringified integers instead of actual integers
  - Stringified booleans (sends "1" and "0" as strings, not as boolean types)
- Approximately 30+ customers still on this version
- Resistant to upgrades (on-premise installations)

[Erik Andersson]:

> For FC enterprise, because it was a very, very interesting CRM system to integrate with, for example, if you don't. If you have a date field in the in the CRM but you don't add any data to it, they set a default date of 1800 ninety-nine in December but. 18199 in December like that is a legit date as far as abscess is concerned. So suddenly suddenly you started having like MA flows that could trigger and all customers were now suddenly 100 years. Years old because FC enterprise provided like their default data. They also have string. They also have integer fields that they say like, yeah, we totally support integer fields. The problem is that they don't send us integers, they send us us stringified integers. They also don't send booleans in their boolean field. They send us one and 0, but they also don't send us one and 0. They send us one and 0 stringified.

### Efficy Enterprise 12.1 (New Version)

- **Generic connector** implementation
- CRM handles all data type conversion and validation
- New customers get this version by default
- Existing 12.0 customers can migrate if consultancy team manages the process
- All outbound features (including outbound mappings) should be supported

[Erik Andersson]:

> There the connector code does all of this black magic transformation but. They have also built a new version of enterprise which does support general connector which is 12.1 and if the customer installs that one then we we don't do any of that transformation then enterprise handles. All of that and it's it's a thing of beauty.

**Known issue:** Outbound mappings UI not visible in 12.1 (bug to be fixed).

---

## Partner Integration Strategy and Lessons Learned

### Why Building Integrations In-house Is Unsustainable

The company learned hard lessons from building legacy connectors. [Erik Andersson]:

> It is more important to find someone who wants to build a connector to Appsys and also own the customer and the support and in exchange. Appsys will get the well cost for Appsys of course. Whatever the customers are using because the amount of time you can spend on debugging and supporting this and the cost you need to spend to fix these bugs. Quite it. It can eat up any revenue you had quite fast if you're unlucky. For example, like I think it costs 2500 crown per hour. To get the old Microsoft Dynamics plugin fixed and like, that's not the cost that is sustainable.

### Recommended Partnership Model: The SiteShop Example

**SiteShop** represents the ideal partnership structure:

1. **SiteShop owns the customer relationship** (not Apsis)
2. **SiteShop charges the customer** for the integration
3. **SiteShop maintains the connector** on their side
4. **Apsis gets compensated** through usage (customer pays per profile imported, per email sent)
5. **SiteShop owns support** (customers don't contact Apsis directly)

[Erik Andersson]:

> The way it works with Siteshop is that it is not apsis that owns the customer, so to say. It is Siteshop that owns the customer, so Siteshop. Um charges the customer for the use of the integration and apps is oh sorry. Appsys benefits from this because Siteshop is losing customers to Appsys which they they then pay for all the profiles that they that they import and all the emails that they send.

### Guidance for Product and R&D

When approached about building a new integration:

> You you should, in all honesty, refuse to do it otherwise. ... Like no find someone that wants to build an implementation for the generic connector and support it and then you would you would be more than. I'm happy to to like add the configuration in apps is required and of course assist with the development of this connector like that we can never get away from, but that is a completely different other thing.

**Strict guidance:** Do not volunteer R&D time to build and maintain a connector. Instead:
1. Define the generic connector specification
2. Invite CRM developers to build against it
3. Provide configuration support and debugging assistance
4. Let the CRM own the connector code and maintenance

---

## Technical Architecture: Justin Platform

The platform that hosts all connectors is called **Justin** (the platform name), while the team is called **Integration** (the team name).

[Erik Andersson]:

> Justin is the name of the platform, so everything I say here is connected to Justin because like when the the team name is integration but the platform name is Justin. So each connectors is handled inside of Justin. You can install them inside of Justin.

All three connector types (legacy, generic, third-party) are installed and managed through Justin.

---

## Key Takeaways

1. **Three connector types exist for historical and pragmatic reasons:**
   - **Legacy connectors:** First-generation, built in-house, now in maintenance-only mode
   - **Generic connector:** Modern standard, shifts responsibility to CRM developers, vastly more sustainable
   - **Third-party integrations:** External companies build and maintain, Apsis provides minimal UI wrapper

2. **The generic connector is the future.** All new integrations should be generic connector implementations.

3. **Legacy connectors have unsustainable economics.** Even at €2,500/hour, supporting and debugging legacy connectors eats revenue quickly.

4. **Consent synchronization is bidirectional and critical.** Misaligned consent between Apsis and CRM can lead to GDPR violations and customer issues.

5. **Apsis should avoid building connectors.** Partner with CRM vendors who will own the customer relationship, maintenance, and support.

6. **Outbound mappings solve the form submission problem.** They allow Apsis to enrich form submission data with mapped attributes so CRMs can populate lead records accurately.

7. **Feature flags on CRMs drive UI visibility.** Apsis should rely on CRM-declared capabilities rather than account-level feature flags for instance-specific features.

8. **Data type enforcement prevents subtle bugs.** Generic connectors enforce strict type contracts, preventing the stringification and type coercion issues that plagued legacy connectors.

---

## Unresolved Questions and Action Items

1. **Event Tool feature flag cleanup:** Remove the account-level feature flag for Event Tool syncing and rely entirely on the CRM's `can_sync_event_tool_activities` capability declaration.

2. **Efficy Enterprise 12.1 outbound mappings:** Investigate why the outbound mappings UI is not visible in the 12.1 connector configuration, despite theoretical support.

3. **Full sync vs. delta sync:** Not covered in this session; delta sync mechanics for connectors to be covered in a future outbound flow session.

4. **Repository structure and code walkthrough:** Planned for a separate session; Erik mentioned this would be the next logical topic.

5. **Detailed outbound flow with logs:** Erik offered to show actual API request/response logs to demonstrate the difference between outbound enabled and disabled states.

---

## Related Topics for Future Sessions

- Inbound flow (full sync and delta sync mechanics)
- Outbound flow (form submissions, event tool, consent syncing details)
- Code repository structure and key folders
- Debugging integration issues using logs
- Setting up a new generic connector (walkthrough with example CRM)