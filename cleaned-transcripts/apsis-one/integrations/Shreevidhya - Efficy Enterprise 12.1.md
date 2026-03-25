---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Efficy Enterprise 12.1 Setup, Generic Connector Architecture, Delta Sync and Full Sync Mechanisms, Sync Conditions, Field Mappings, Subscription Mappings, Queries and Profiles, Bidirectional Consent Sync, Logging and Debugging with CloudWatch]
speakers: [Shreevidhya Ganesan, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Delta Sync Manager, Delta Sync Worker, Field Mappings, Subscription Mappings, Sync Conditions, Queries and Profiles, CloudWatch Logs, Efficy Enterprise 12.1 CRM]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session demonstrated the practical setup and operation of the **Efficy Enterprise 12.1** integration using the **Generic Connector** architecture. The team walked through installing the connector in a new section, configuring field and subscription mappings, testing delta sync and full sync operations, implementing sync conditions, and working with queries and profiles. Key emphasis was placed on understanding the bidirectional consent sync, delta sync mechanics via webhooks, and how to debug issues using CloudWatch logs.

---

## Generic Connector and Efficy Enterprise 12.1 Installation

### Setup Overview

The Generic Connector requires two URLs:
- One for the **CRM** (external Efficy Enterprise system)
- One for the **installation** (Apsis One environment)

[Shreevidhya Ganesan]: The external environment and API key are provided by the CRM team. The installation process leverages these credentials to establish the connection.

### Section Limitation

[Shreevidhya Ganesan]: **One CRM installation per section** — this is a deliberate architectural limitation discussed in previous sessions. However, multiple sections can each have their own installation of the same CRM system, allowing exploration across different sections.

### Creating Named Sections for Organization

To maintain clarity during testing, name each integration descriptively (e.g., `EE 12.1`) so developers can easily identify which installation exists in which section.

---

## Field Mappings and Auto-Mapping

### Mapping Process

Once installation is complete, navigate to **Field Mappings** and use the **auto-mapping** feature to automatically detect and map available fields from the external system to Apsis One. This serves as a starting point; manual adjustments can be made afterward.

### Testing Record Creation

To verify the connector is functioning:
1. Create a new **contact** (note: called "profiles" in some contexts) in the external Efficy Enterprise system
2. Populate required fields with valid data
3. Phone numbers must follow proper international format (e.g., `+46700...`)
4. Observe the newly created contact appears in Apsis One after sync completes

[Shreevidhya Ganesan]: Creating records directly in the external system is faster and more reliable than triggering a full sync initially.

---

## Delta Sync: Real-Time Synchronization

### How Delta Sync Works

Delta sync is **real-time** and begins automatically once the connector is installed. It operates via **webhook callbacks**:

1. During installation, Apsis One registers a **callback URL** with the external CRM system
2. The mapped fields are sent to the external system so it knows which fields to monitor for changes
3. Whenever a change occurs in any mapped field on a profile in the external system, the CRM triggers the callback URL with the profile ID
4. Apsis One receives the webhook and processes the change through the **Delta Sync Manager** → **Delta Sync Worker** pipeline

[Shreevidhya Ganesan]: For Efficy Enterprise, delta sync can take a few minutes due to batch processing — the external system collects changes and sends them in short batches rather than individually.

[Tomasz Kowalski]: Delta sync is triggered by changes in the external system; webhook callbacks notify Apsis One of which records have changed.

### Delta Sync Logs

Logs are stored in CloudWatch with structure:
```
Integration Key: {account_id}/{section_id}/{integration_name}
Message Group ID: {integration_id}
Entry ID: {crm_id}
```

Delta Sync Manager produces messages that are consumed by Delta Sync Worker for processing. Validation failures (e.g., phone number format errors) are logged in Delta Sync Worker logs and will skip that particular field update.

---

## Subscription Mappings

### Creating Subscriptions

Subscriptions in Apsis One must be explicitly created before they can be mapped to external system profiles:

1. Create a **folder** in the Subscriptions section (e.g., "enterprise 12.1")
2. Create a **subscription** within that folder (e.g., "profile QA")
3. In the integration settings, map external profiles to these subscriptions under **Subscription Mappings**

### Mapping Process

In **Subscription Mappings**, link external system profiles to Apsis One subscriptions. For example, map "profile QA" from the external system to the locally created "profile QA" subscription.

[Shreevidhya Ganesan]: Only mapped subscriptions will be synced. Unmapped external profiles will not create subscriptions in Apsis One.

---

## Bidirectional Consent Synchronization

### Consent is Bidirectional; Attributes Are Not

**Critical distinction**: 

- **Attributes** (name, email, phone, etc.) sync **inbound only** — from external system to Apsis One
- **Consent/Subscription status** syncs **bidirectionally** — changes in either system are reflected in the other

### Consent State Model

A profile can have three consent states:
1. **No subscription** (never opted in)
2. **Opted in** (subscribed)
3. **Opted out** (unsubscribed)

### Testing Bidirectional Consent

**Inbound (External System → Apsis One):**
1. In Apsis One, navigate to a profile's **Consent Timeline**
2. Change the consent status (e.g., unsubscribe)
3. After a brief delay (batch processing), the change appears in the external system

**Outbound (External System → Apsis One):**
1. In the external system, modify a profile's subscription status
2. The change triggers an **outbound delta sync** in Apsis One
3. The consent state updates in Apsis One

[Shreevidhya Ganesan]: Consent changes trigger the outbound process; attribute changes do not propagate back to the external system.

### Attribute Overwrite Risk

[Lukasz Grabowski]: If an attribute is changed in both systems and delta sync occurs, the external system value will overwrite the Apsis One value for mapped fields.

---

## Sync Conditions: Filtering Which Profiles Are Synchronized

### Purpose and Logic

Sync conditions allow fine-grained control over **which profiles** are included in syncs. All sync conditions use **AND logic** — a profile must satisfy **all** conditions to be synced.

### Condition Types

Sync conditions can reference:
- **Mapped attributes** from the external system
- **Values** to match against (string, boolean, etc.)
- **Operators**: currently supports equality (`=`); enhanced operator support like `contains` exists but is **behind a feature flag** pending external system support

### Example Condition

```
Email = [user@example.com]
```

Only profiles with this exact email address will be synchronized.

### Sync Condition Behavior

- Applied to **all sync types**: delta sync, full sync, and real-time sync
- If a profile **does not match** the condition, it is **skipped** during sync
- If a **delta sync message** arrives for a profile that doesn't match the condition, that message is **thrown away** (not processed)

[Shreevidhya Ganesan]: Sync conditions are evaluated **on the Apsis One side**, not pushed to the external CRM. The full dataset is queried from the external system, but filtering occurs locally before import.

### Full Sync Report with Conditions

When a full sync completes with active sync conditions:
- **Successfully synced**: profiles matching all conditions
- **Skipped**: profiles not matching one or more conditions
- The report displays counts for both

---

## Full Sync vs. Delta Sync: When to Use Each

### Delta Sync Limitations

Delta sync only synchronizes **changes** that occur after installation. It does **not** bring in historical data or profiles that existed before the connector was installed.

### Full Sync Purpose

**Full sync brings existing data into alignment**:
- Pulls **all profiles** from the external system
- Applies sync conditions to filter which profiles to import
- Establishes a consistent baseline where both systems have the same profiles and attributes
- **Prerequisite**: Before relying on delta sync, perform at least one full sync to ensure data parity

[Shreevidhya Ganesan]: Full sync is required primarily the first time after installation. Thereafter, delta sync maintains ongoing synchronization. Full sync can be re-run later if needed to reconcile data.

---

## Queries and Profiles: Tagging Profiles via External System Queries

### Concept

**Queries** are pre-built filters or segments in the external CRM system (e.g., "VIP customers", "high-value accounts"). Apsis One can import profiles matching these queries and automatically tag them.

### How It Works

1. In the external system, define a query (e.g., "apsis_test")
2. In Apsis One, navigate to **Queries and Profiles**
3. Select a query and optionally enable **Recurring** (runs daily via cron job)
4. Click **Start Import** to immediately import profiles matching that query
5. Imported profiles are tagged with the query name (e.g., "apsis_test" tag)

### Use Case

Tags can be leveraged in **segmentation** to filter profiles. For example:
- Import all "VIP" profiles via a query
- Tag them with the "VIP" tag
- Create a segment filtering on the VIP tag for targeted campaigns

[Shreevidhya Ganesan]: Queries and profiles are **not** updated by full sync; they are managed independently. Recurring imports keep the tagged profile list in sync with external system queries.

[Lukasz Grabowski]: When does tagging occur? After running the query import, profile tags appear immediately and persist in the audience.

---

## Debugging and Log Access

### CloudWatch Log Structure

All connector logs are in **CloudWatch** on the stage environment. Key log groups:
- **Delta Sync Manager**: produces formatted messages for worker consumption
- **Delta Sync Worker**: processes messages, logs validation errors and transformations

### Finding Logs by Integration

Use the **Integration Key** as the search filter:
```
{account_id}/{section_id}/{integration_name}
```

Then search by **Message Group ID** for a specific integration instance, and **Entry ID** (CRM ID) for a particular profile.

### Common Log Patterns

**Validation failures** appear in Delta Sync Worker logs with descriptive messages:
```
tried to validate and format phone number. That's the reason.
mobile number is too short
```

[Shreevidhya Ganesan]: Fields that fail validation are skipped; the error does not halt the entire profile update. Other fields in the same profile still sync.

### Log Navigation Tips

- Adjust time filters (past 5-10 minutes) to find recent activity
- Look for **webhook request parameters** in Delta Sync Manager logs to see incoming webhook payload
- Search for field names (e.g., "mobile") to trace specific attribute updates

---

## Multiple Instances and Scaling Limitations

### Efficy Enterprise: Multiple Installations Supported

Efficy Enterprise **can** have multiple independent installations across different sections in the same account. Each installation can have its own API key and configuration.

[Shreevidhya Ganesan]: This flexibility allows parallel testing and exploration of different configurations.

### Other CRM Systems: Single Instance Per Account

For **Microsoft Dynamics**, **Tribe**, **E-deal (Efficy Corporate)**, and **Zapier**:
- **Only one installation per section** 
- The reason relates to a feature called **deep linking** — the ability to open a profile in the external system directly from Apsis One

[Shreevidhya Ganesan]: Deep linking requires a unique, consistent mapping between Apsis One profiles and external system profiles, which is why multiple installations are not supported for these systems.

---

## Testing Setup and Environment Access

### API Keys and URLs

For testing different CRM integrations:
- **Efficy Enterprise, Tribe, E-deal**: API keys and test instance URLs are stored in **LastPass**
- Each external CRM system provides a **test/sandbox environment**
- Copy the API key and base URL from LastPass and paste into the installation form

### Generic Connector Advantage

Since the Generic Connector shares the same underlying code across all integrations:
- **One working installation is sufficient** to test features and validate the connector logic
- New external systems that adhere to the **standard API specification** can be integrated without code changes
- A **Postman collection** is available for testing connector endpoints directly

[Shreevidhya Ganesan]: To test any integration, you only need the API key, base URL, and the external system's test instance. All endpoint interactions can be verified with Postman before UI testing.

---

## Adding New External Systems

### Generic Connector Approach

The team has moved to a **standardized Generic Connector approach**:
- Any external system that adheres to a defined API specification can be integrated
- No custom connector code is required
- The specification is consistent and straightforward

### Onboarding a New System

When a new integration request arrives:
1. Verify the external system supports the **Generic Connector specification**
2. Provide the customer with the specification document and requirements
3. The external system provides API key and test environment
4. Install using the generic connector (same UI/process as any other system)
5. Test via Postman and UI

[Shreevidhya Ganesan]: Once you understand the architecture, you can confidently handle integration requests. The Postman collection makes it easy to test endpoints without building UI interactions first.

---

## Key Takeaways

1. **Delta sync is automatic and webhook-driven** — changes in the external system trigger callbacks to Apsis One, but with a slight delay for batch processing.

2. **Full sync is essential for baseline data parity** — before relying on delta sync, run at least one full sync to import existing historical data.

3. **Sync conditions are evaluated locally** — the external system is queried fully, but filtering happens in Apsis One; all conditions must match (AND logic).

4. **Consent is bidirectional; attributes are not** — subscription status changes sync both ways, but attribute changes only flow inbound. Be aware of overwrite risks.

5. **Field validation skips bad data** — phone numbers, emails, and other fields are validated. If validation fails, that field is skipped; the rest of the profile still syncs.

6. **Queries and profiles create tags** — queries from the external system can be imported to tag profiles for segmentation purposes.

7. **CloudWatch logs are searchable by Integration Key** — use account ID, section ID, and integration name to locate logs for specific connectors.

8. **Multiple Efficy Enterprise instances are supported; others are limited to one per section** — deep linking is the architectural constraint for single-instance systems.

9. **Generic Connector standardizes new integrations** — any system supporting the specification can be added without custom code.

10. **Postman collection enables offline testing** — test connector endpoints directly before attempting UI workflows.

---

## Unresolved Questions and Action Items

- **API key creation in Efficy Enterprise 12.1**: The team did not have permissions to demonstrate creating a new API key in the external CRM. This should be explored with the enterprise team if needed for future integration setup.

- **Query building**: Shreevidhya has not personally created queries in Efficy Enterprise and relied on existing queries. Eric can provide deeper guidance on query creation tomorrow.

- **Feature flag: Enhanced sync condition operators** — The `contains` operator for sync conditions is behind a feature flag pending external system support. Timeline and requirements for enabling this feature are TBD.

- **Depth dive into sync condition logic**: Eric will cover the complete sync condition logic and edge cases in a follow-up session.

- **Workshop for new external system onboarding**: Lukasz requested a future workshop covering the end-to-end process of integrating a brand-new external CRM system using the Generic Connector specification. Shreevidhya indicated this is possible once foundational architecture knowledge is solid.