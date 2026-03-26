---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Generic Connector Implementation, Field Mappings, Delta Sync vs Full Sync, Sync Conditions, Consent Management, Query and Profile Tagging, Multi-section Installation, Webhook Integration, CloudWatch Logging]
speakers: [Shreevidhya Ganesan, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [FEC Enterprise 12.1, Delta Sync Manager, Delta Sync Worker, CloudWatch Logs, Generic Connector, Subscription Mappings, Sync Conditions, Full Sync, Queries and Profiles]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This knowledge transfer session covers a hands-on walkthrough of setting up and testing the **Efficy Enterprise (FEC) 12.1** integration using Apsis One's **generic connector** architecture. The session demonstrates key integration workflows including field mappings, real-time delta synchronization, consent bidirectionality, sync conditions (filtering rules), and query-based profile tagging. The team tested profile creation, subscription state changes, logging inspection in CloudWatch, and multi-section deployment patterns. Notable discussions included the distinction between delta sync (real-time, webhook-driven) and full sync (historical data pull), validation failures, and the operational difference between legacy connectors and the standardized generic connector approach.

---

## Generic Connector Architecture vs Legacy Connectors

**Shreevidhya [Presenter]**: This is a better environment and it's just a generic connector, not the legacy. 

The generic connector represents a standardized interface that external CRM systems must implement, as opposed to the older **legacy connectors** which were built in-house by Apsis. The generic connector approach enables easier onboarding of new external systems because they must adhere to a standard specification rather than requiring custom development for each system.

### Two URLs Configuration Pattern

> As usual, we have like 2 URLs, one for the CRM and one for our installation.

The setup requires:
- **CRM URL**: The external system endpoint (e.g., FEC Enterprise instance)
- **Apsis One Installation URL**: Our system's endpoint for the integration

The API key and base URL must be provided by the CRM environment. In this case, Shreevidhya noted: "it was like the environment which was provided by the CRM so" — the team doesn't have permissions to create new integrations in the customer's FEC instance; those credentials are pre-configured and shared via LastPass or similar secure channels.

### Multi-section Limitation

A critical limitation exists: **only one instance of a connector per section**. However, this limitation is specific to certain systems:

- **Efficy Enterprise**: Can have multiple installations across different sections in the same account
- **Microsoft Dynamics, Shopify**: Cannot have multiple installations in the same section (one instance maximum per CRM type)

**Rationale for the Dynamics/Shopify limitation**: A feature called **deep linking** allows direct navigation from an Apsis One profile to its corresponding record in the external system. This feature breaks if multiple instances exist because the system cannot determine which external system instance to link to.

---

## Installation Setup and Field Mapping

### Creating the Integration

The installation process involves:
1. Navigating to the integrations section
2. Entering the CRM URL and API key
3. Naming the integration (e.g., "EE 12.1" for Enterprise 12.1) for easy identification
4. Triggering auto-mapping of fields

**Shreevidhya**: You can have like if you say enterprise 12.1 name so it will be easy for you to identify what installation you have in that section.

### Field Mapping Process

After auto-mapping, the common mapped fields included:
- `Email` → Email
- `Title`
- `Language`
- `Banner`
- `Mobile` (phone number)

**Important caveat on phone number validation**: When creating test contacts, phone numbers must follow proper international format (e.g., `+46700...`). If validation fails, the field will be silently skipped during sync processing.

---

## Delta Sync vs Full Sync: Operational Differences

### Delta Sync (Real-Time, Webhook-Driven)

Delta sync operates continuously after installation and is **triggered by webhooks** when changes occur in the external system.

**How it works**:
1. During installation, Apsis One registers a **callback URL** with the external CRM
2. When any mapped field changes in the external system, the CRM sends a webhook notification containing the profile ID
3. The callback is handled by the **Delta Sync Manager** service
4. Messages are produced and queued for processing

**Processing delay**: Different CRMs have different batch collection windows. FEC Enterprise takes "couple of minutes" for batch processing because it collects changes and sends them in batches, unlike Microsoft Dynamics which triggers immediately.

### Full Sync (Historical Data Pull)

Full sync is required initially to bring existing data into alignment:

> Full sync maybe like you know it's required for the first time because you want to pull all the records from the external system, right?

**When to use full sync**:
- First-time installation to import existing profiles from the external system
- After adding new mapped attributes or profiles in the external system where no changes have occurred
- To re-synchronize data that may have drifted

**Key difference**: Delta sync only captures **changes**. If a profile existed in the external system before the field was mapped, delta sync won't retrieve it. Full sync brings those dormant records into Apsis One.

---

## Field Synchronization: Unidirectional vs Bidirectional

### Attributes: One-Direction Only (CRM → Apsis)

Regular profile attributes flow **only from the external system to Apsis One**. If you map a field like "middle name":

**Shreevidhya**: if you have mapped it, so whatever the fields you have mapped, only those fields will be reflected.

**Critical gotcha**: If you edit an attribute in Apsis One and then that same attribute changes in the external system, the external system's value **will overwrite** the Apsis One value during the next delta sync.

```
Apsis One (edit) → [overwritten by] ← Delta Sync from External CRM
```

### Consent: Bidirectional Sync

Unlike attributes, **subscription consent states sync bidirectionally**:

- **Inbound**: When a subscriber opts out in the external system, the webhook triggers, and their subscription state in Apsis One changes to opted-out
- **Outbound**: When you change a subscription state in Apsis One (e.g., from opted-out to opted-in), the change is pushed back to the external system

**Subscription states**:
- No subscription (default)
- Opted in (subscribed, can receive messages)
- Opted out (unsubscribed)

---

## Webhook Mechanism and Registration

### Callback URL Registration During Installation

> When we do a registration we have, we will be sending the fields whatever we have mapped in our system to the external system. So the so that the external system will be knowing that these are the fields which access has been mapped for. And any changes in that field, the web hook will be triggered.

**Process**:
1. Apsis One provides a list of mapped fields to the external system during installation
2. The external system registers this list internally
3. When changes occur to **any** of those mapped fields, the external system triggers the webhook
4. The webhook endpoint lives in the **Delta Sync Manager** service
5. The payload includes the profile ID and the nature of the change

**Endpoint format**: 
```
[Apsis installation URL]/delta-sync-endpoint
```

---

## Debugging with CloudWatch Logs

### Log Structure and Key Identifiers

All integration activity is logged in **CloudWatch** on the staging environment. There are two main log groups:

1. **Delta Sync Manager**: Produces messages in the format our worker can process
2. **Delta Sync Worker**: Performs actual data transformation and writes to Apsis One

**Log grouping**: You can search for logs by constructing an **integration key**, which includes:

```
account_id/section_id/integration_name:crm_id
```

Example from the session:
```
integration_key: [account]/[section_id]/enterprise:179
```

Where:
- `account`: Account identifier
- `section_id`: Section within the account
- `enterprise`: Integration name (readable identifier)
- `179`: CRM ID (the profile ID in the external system)

**Message group ID**: Also searchable and helps correlate related log entries

### Examining Delta Sync Worker Logs

The Delta Sync Worker logs reveal validation errors. In this session, when a mobile phone number failed validation:

```
[Webhook request] parameters: fields, records
  mobile: [value]
  phone_to: [value]
  
[Delta Sync Worker] WARNING
  "Tried to validate and format phone number. Mobile updating the attribute mobile with delete status probably just failed the validation."
  
[Reason] "All number is too short"
```

**Key takeaway**: If a field fails validation during processing, it is **silently skipped** — the sync continues without error, but that field is not written to Apsis One. Always check the logs if expected data is missing.

---

## Sync Conditions: Filtering Profiles During Sync

### Purpose and Behavior

Sync conditions allow you to filter which profiles are synchronized from the external system to Apsis One. This filtering happens **on the Apsis One side**, not on the CRM side.

> No, in our side, in our side, it's not with the CM. [Speaker clarifying when asked where filtering occurs]

### Multiple Conditions Use AND Logic

When multiple sync conditions are defined, **all conditions must be satisfied** for a profile to sync:

**Example from session**:
- Condition 1: `Email` contains `@lucas.com`
- Condition 2: `Name` equals `Lucas`

Result: Only profiles matching both conditions sync. If a profile's name changes to something else (e.g., "Mikhail"), that profile will no longer sync even if the email still matches.

### Full Sync Respects Conditions

When a full sync is triggered, it respects all active sync conditions:

```
profiles in external system: 90
profiles matching sync conditions: 1
full sync result: 1 successfully synced, 89 skipped
```

### Supported Operators

From the session UI:
- `equals` (standard)
- `contains` (marked as an "enhanced feature" behind a feature flag, requires CRM support)

### Use Case

Sync conditions are useful for:
- Testing with a subset of profiles before rolling out to all
- Excluding certain profile types (e.g., test records)
- Incremental rollouts to large customer databases

---

## Queries and Profiles: Dynamic Tagging

### Concept

A **query** in the external system is a saved filter or segment (analogous to a "smart list"). Apsis One can import profiles matching a query and **automatically tag them** with the query name.

**Shreevidhya's explanation**:
> It's like a kind of you can how how to imagine that is say for example this feature comes in handy when you have you want to tag profile say there will be like a query in the external system where you can say the profiles which which are in VIP status or something. So and now when you try to import those profiles and have the profiles imported. You will be having those profile tagged as VIP so that in segmentation it can become handy to just filter out the profiles which has the tag VIP.

### Setup Process

1. Navigate to **Queries and Profiles** section in the integration configuration
2. Select an existing query from the external system (e.g., "APSIS test")
3. Optionally enable **recurring import** with a cron schedule (e.g., daily)
4. Click **Start Import**

### Import Execution

- Status progresses: "Calculating total" → processes profiles → completion
- The system reads the external query, fetches matching profiles, and tags each with the query name in Apsis One
- Example: 5 profiles from the "APSIS test" query were imported and tagged with the tag `FS enterprise` (the query name)

### Profiles Appear as "Unknown"

When first imported via a query, profiles appear without attribute data in Apsis One (showing only as numbered records). Attribute data populates after a **full sync**, which fills in fields like name and email.

### Difference from Full Sync

- **Full Sync**: Pulls all profiles from the external system based on sync conditions; no tagging
- **Queries/Profiles Import**: Pulls profiles matching a specific saved query; automatically tags them; can be scheduled to recur

---

## Subscription Mappings

### Creating Subscriptions in Apsis One

Subscriptions are created in the **Subscriptions** folder in the Apsis One audience section. Once created, they can be mapped to external system profiles:

1. Create a subscription folder (e.g., "Enterprise 12.1")
2. Create a subscription within it (e.g., "Profile QA")
3. In the integration settings, map external profiles to this subscription
4. Set the mapping delivery method (e.g., Email, SMS)

### Consent State Tracking

When a profile is synced with a subscription mapping:
- The system records the initial subscription state (opted in, opted out, or none)
- The **consent timeline** shows when consent changes occurred
- Both inbound (external → Apsis) and outbound (Apsis → external) changes are tracked

---

## Architecture and Testing Workflow

### Testing Multiple Integrations

**For Efficy Enterprise and similar systems**:
- Multiple installations can exist in different sections of the same account
- Use LastPass to retrieve pre-configured API keys and URLs for test environments
- Each external CRM vendor provides their own test instance with shared credentials

**For single-instance systems** (Dynamics, Shopify):
- Only one installation per account
- Test with the single shared test instance
- All changes are visible to the team

### Postman Collection for Manual Testing

For developers wanting to test the generic connector specification without a full UI:
- A **Postman collection** is available with pre-built API requests
- Requires only the external CRM's API key and base URL
- Allows testing of all generic connector endpoints (create profiles, update subscriptions, etc.)
- Useful for understanding the standardized interface before implementing a new CRM

---

## Onboarding a New External System: The Generic Connector Path

### Standard Specification Approach

Instead of building custom legacy connectors, new external systems must:

> Will be having like a set of standard specification which they have to adhere to.

1. Implement the **generic connector specification**
2. Provide webhook support for change notifications
3. Expose standard CRUD endpoints for profiles and subscriptions
4. Support field mapping configuration via API

### Development Workflow for New Integrations

When a customer requests a new external system:
1. Share the generic connector specification with the external system's development team
2. The team implements the standard interface
3. Provide Postman collection and documentation
4. Once implemented, the same Apsis One generic connector code handles all systems
5. Testing occurs on the external system's test instance

**Advantage**: Decouples Apsis One development from individual CRM implementations. The integration code path is identical for all generic connectors.

---

## Key Takeaways

1. **Generic Connector is Standardized**: Unlike legacy in-house connectors, generic connectors use a standardized interface that external systems implement. This approach scales better and reduces maintenance burden.

2. **Delta Sync is Webhook-Driven and Real-Time**: After installation, changes in the external system trigger webhooks that fire immediately (or in batches for FEC). This is the continuous sync mechanism. No fields need to be manually synced.

3. **Full Sync Brings Historical Data**: Full sync is necessary to import profiles that existed before field mappings were created. It also respects sync conditions, allowing filtered imports.

4. **Attributes are One-Direction Only**: Profile attributes flow from the external system to Apsis One. Changes in Apsis One are overwritten by external updates during the next sync. Only **consent states are bidirectional**.

5. **Sync Conditions Filter on Apsis Side**: Filtering happens within Apsis One after receiving webhook data, not at the CRM. All conditions must be satisfied (AND logic).

6. **Phone Number Validation is Strict**: Invalid phone numbers silently fail validation and are excluded from sync. Always check CloudWatch logs for validation errors.

7. **Queries Tag Profiles Dynamically**: Importing profiles via a saved external query automatically tags them in Apsis One, enabling segment-based filtering and campaigns.

8. **Multi-section vs Single-instance Limitation**: Efficy Enterprise allows multiple installations across sections. Dynamics and Shopify allow only one per account due to deep-linking constraints.

9. **CloudWatch Logs Use Integration Keys**: Search logs using the pattern `account/section/integration:crm_id` to find relevant entries. Check Delta Sync Worker logs (not just Manager) for validation errors.

10. **Testing Infrastructure is Shared**: Test credentials for external systems are stored in LastPass and reused across the team. For single-instance systems, one shared test environment suffices.

---

## Unresolved Questions and Action Items

1. **How to Create Queries in FEC**: Shreevidhya noted she has not personally built queries from scratch in FEC Enterprise — only used existing ones. **Action**: Eric to cover query building in future deep-dive session.

2. **API Key Creation Permissions in FEC**: The team lacks permissions to create new integrations/API keys in the customer's FEC instance. **Action**: Ask the enterprise team about the process if needed.

3. **Profile Attribute Details in FEC vs Apsis**: Some profiles imported via query contained only email, not first/last name fields. Unclear if this is a mapping issue or FEC data limitation. **Action**: Eric to review the complete field-mapping logic.

4. **Workshop on New System Onboarding**: Lukasz requested a hands-on workshop covering the full workflow for onboarding a brand-new external system (architecture review, spec implementation, testing). **Action**: Schedule with Shreevidhya and/or Eric once fundamentals are solid.

5. **Consent Timeline Delay**: After opt-in change in Apsis One, the consent timeline in the external system did not update immediately. System was still processing at end of session. **Action**: Monitor and confirm eventual consistency; clarify acceptable sync latency thresholds.