---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Efficy Enterprise 12.1 Integration Setup, Field Mappings, Delta Sync and Full Sync, Sync Conditions, Webhook Callbacks, Consent Synchronization, Queries and Profiles Tagging, Generic Connector Architecture, CloudWatch Logging]
speakers: [Shreevidhya Ganesan, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Efficy Enterprise 12.1 CRM, Delta Sync Manager, Delta Sync Worker, Field Mappings, Sync Conditions, Webhook Callbacks, CloudWatch Logs, Audience Profiles, Segments, Queries and Profiles]
session_type: knowledge-transfer
subdomains: [Efficy Enterprise 12.1 Integration, Architecture]
---

## Session Overview

This knowledge transfer session covers the practical setup and operation of the Efficy Enterprise 12.1 integration within the Apsis One platform using the generic connector. The team walked through installation configuration, field mapping, delta sync mechanics, sync conditions, consent synchronization, and logging/debugging approaches. The session includes hands-on demonstrations of creating test integrations, triggering syncs, and understanding bidirectional consent flows between systems.

---

## Generic Connector Architecture and Overview

**Shreevidhya Ganesan** explained that the team has moved to a **generic connector approach** for integrations. This means any external system that wants to connect with Apsis One must adhere to a set of standard specifications. The generic connector is not a legacy system and provides better performance and more standardized functionality.

[Lukasz Grabowski]: Questioned how onboarding a new external system would work operationally.

[Shreevidhya Ganesan]: Clarified that customers requesting a new external system integration receive a specification document outlining the standard interface/API requirements. Once the external system adheres to the specification, integration becomes straightforward. A **Postman collection** is available for testing all endpoints—requiring only the API key and prefixed CRM URL.

> The generic specification is "more standardized and straightforward" compared to legacy approaches. Development teams can test different features independently using Postman without deep architectural understanding first.

---

## Installation and Configuration

### Setting Up Efficy Enterprise 12.1 Integration

Installation requires two URLs:
- One for the CRM system
- One for the Apsis One installation environment

[Shreevidhya Ganesan]: Noted a **limitation**: Only one CRM installation per section is possible (discussed and documented the previous day).

Setup process:
1. Navigate to a new section (or create one)
2. Name the section descriptively (e.g., "EE 12.1" for Efficy Enterprise 12.1) for easy identification
3. Create a new integration link using the CRM's environment and API key
4. The API key and environment are provided by the CRM administration team; developers typically do not create these themselves

[Michal Rosikiewicz]: Asked where and how to create API keys in FEC 12.1 CRM.

[Shreevidhya Ganesan]: Responded that the test environment and API keys are pre-provisioned; the team uses what is provided rather than creating new integration credentials.

---

## Field Mappings and Auto-Mapping

After installation, **field mappings** are configured to define which Apsis One contact fields sync with external CRM fields.

Process:
1. Navigate to **Field Mappings** section
2. Use **auto-mapping** feature to automatically map common fields (when available)
3. Save mappings

[Lukasz Grabowski]: During the session, created a test contact with fields: name, email, country, title, language, banner, mobile, etc.

Mapping example created:
- Email → Email
- Name → Name
- Mobile → Mobile

Note: Fields not mapped (e.g., Skype) are ignored. Phone numbers must follow a valid format (e.g., `+46700...`).

---

## Delta Sync: Real-Time Synchronization

### How Delta Sync Works

Delta sync is **real-time synchronization** that starts automatically once installation is complete. It monitors changes in both systems and propagates them.

**Webhook-based callback mechanism:**

When the integration is installed, Apsis One registers a **callback URL** with the external CRM system. Whenever a change occurs in the CRM, the system triggers the webhook with the profile ID that changed.

[Shreevidhya Ganesan]: Explained the registration process—during installation, mapped field information is sent to the external system so it knows which fields to monitor.

> "So whenever there is a change which is happening in the external system CRM, the CRM tries to notify back to the integration by triggering the callback URL with the ID of the profile for which the changes has happened."

### Full Sync vs Delta Sync

- **Delta sync** runs continuously and handles incremental changes
- **Full sync** is required initially to pull all existing records from the external system and bring both systems into alignment
- After full sync, delta sync maintains synchronization for new or modified records

[Shreevidhya Ganesan]: Emphasized that full sync is essential for the first-time setup because it ensures existing data from the external system is imported.

### Delta Sync Timing and Performance

Delta sync latency varies by system:
- **Dynamics**: Immediate (happens synchronously)
- **Efficy Enterprise 12.1**: Takes a few minutes due to batch processing. The external system collects changes and sends them in batches.

[Lukasz Grabowski]: Asked how long delta sync typically takes.

[Shreevidhya Ganesan]: Explained the delay is due to batch processing on the Efficy side—changes are collected and sent in short batches.

---

## CloudWatch Logging and Debugging

### Finding Logs

Logs are available in **CloudWatch** on the staging environment.

**Two primary log groups for delta sync:**

1. **Delta Sync Manager** (`log group: Delta Sync Manager`)
   - Shows webhook request parameters and the messages produced
   - Contains initial payload information and field details
   - Not the place to find field-level validation errors

2. **Delta Sync Worker** (`log group: Delta Sync Worker`)
   - Shows actual processing and validation logic
   - Contains warnings and errors from field validation (e.g., phone number format failures)
   - Displays if attributes were skipped due to validation failures

### Using Integration Key for Log Search

Logs can be filtered and searched using the **integration key**, which has this format:

```
account_id/section_id/integration_name
```

Additional context in logs:
- `message_group_id` - provides integration ID
- `CRM_ID` - the profile ID in the external system (e.g., profile with CRM ID 179)
- Each log entry shows the specific attribute being synced and any validation warnings

[Lukasz Grabowski]: Observed a validation error in logs where a mobile number failed validation because it was too short. The logs showed the exact failure: `tried to validate and format phone number. That's the reason.`

> Integration key format: `{account_id}/{section_id}/{integration_name}` — this allows precise log filtering in CloudWatch.

---

## Bidirectional Consent Synchronization

### Understanding Consent Direction

**Critical distinction:** Apsis One syncs **attributes unidirectionally** (from external system to Apsis) but **consent bidirectionally**.

- **Attributes**: Changes in Apsis One do NOT sync back to the external CRM. Only changes in the external CRM are reflected in Apsis.
- **Consent**: Changes in either system are reflected in the other.

[Lukasz Grabowski]: Tested this by creating a profile in the external system, then checking if changes to attributes in Apsis (adding a middle name) would sync back to the external system.

[Shreevidhya Ganesan]: Clarified that mapped attributes only sync one direction—external CRM → Apsis. Editing mapped fields in Apsis will be overwritten if the external system changes them.

> "Whatever the fields you have mapped, only those fields will be reflected [from external system to Apsis]. If you change here something and then change it in the external system, it will be overwritten by data from here."

### Consent Workflow Example

The team demonstrated the full consent sync cycle:

1. **Opt-out in Apsis**: A profile (Lucas) was unsubscribed from a subscription (profile QA) in Apsis One
2. **Delta sync propagation**: The change triggered delta sync to the external system
3. **Verification**: After a short delay (a few minutes), the profile appeared as "opted out" in the external FEC system
4. **Reverse opt-in**: The profile was re-subscribed in FEC, which propagated back to Apsis via delta sync
5. **Bidirectional confirmation**: The profile's consent state synchronized in both directions

**Consent states:**
- No subscription: Default state (no record exists)
- Opted in: Active subscription
- Opted out: Unsubscribed

---

## Subscription Mappings

### Creating Subscriptions for Integration

Subscriptions must be created and mapped to external system profile lists.

Process:
1. Create a **folder** in the Apsis subscriptions section (e.g., "Enterprise 12.1")
2. Create a **subscription** within that folder with a meaningful name (e.g., "profile QA")
3. Map external profile lists to these subscriptions

[Lukasz Grabowski]: Created a subscription named "profile QA" mapped to an external profile in FEC.

[Shreevidhya Ganesan]: Confirmed that subscription names should match external system profile names for clarity. Once mapped, profiles can be added to these subscriptions in the external system, which triggers delta sync to bring them into Apsis.

---

## Sync Conditions: Filtering Profiles

### Purpose and Mechanism

**Sync conditions** allow filtering which profiles are synchronized from the external system to Apsis One. Conditions are evaluated in Apsis (not on the external CRM side).

- Sync conditions apply to **both full sync and delta sync**
- If a profile does not satisfy all conditions, it is **skipped** entirely
- Multiple conditions use **AND logic**—all conditions must be satisfied for a profile to sync

### Sync Condition Example

A condition was created: `Email equals [test_email@example.com]`

When full sync was executed:
- 90 total profiles were fetched from external system
- Only 1 profile matched the condition
- 89 profiles were skipped and not imported

[Lukasz Grabowski]: Questioned whether filtering happens on Apsis or in the CRM.

[Shreevidhya Ganesan]: Confirmed it is **Apsis-side filtering**. The external system sends all profiles, then Apsis applies conditions client-side.

### Advanced Sync Conditions (Feature Flag)

An enhanced feature exists that allows `contains` operator (in addition to `equals`), but this is **behind a feature flag** and requires external system support to enable.

Example condition seen:
```
Email equals 'value.@fec.com'
```

The team attempted a `contains` operator but noted it requires platform support.

---

## Queries and Profiles: Dynamic Profile Tagging

### What Are Queries?

In Efficy Enterprise terminology, a **query** is a saved search/filter in the external CRM that identifies a set of profiles matching certain criteria. When integrated with Apsis One, queries can be used to automatically **tag profiles** during import.

Use case: A query in FEC identifies all VIP customers. When imported via Apsis, these profiles are automatically tagged with "VIP" for use in segmentation.

[Lukasz Grabowski]: Described queries as creating tags: *"It creates stock"* / *"It tags the profile."*

[Shreevidhya Ganesan]: Confirmed: *"It tags the profile for a particular... [query]."*

### Importing Profiles via Queries

Process:
1. Navigate to **Queries and Profiles** section in the integration settings
2. Select an existing query from the external system (e.g., "apsis test")
3. Enable **Recurring** option (set to run automatically, e.g., every day) or leave unchecked for one-time import
4. Click **Start Import**

The system calculates the total number of matching profiles and creates a tag in Apsis automatically.

[Lukasz Grabowski]: Ran an import of the "apsis test" query and saw 5 profiles processed successfully.

Result: A new tag "FS Enterprise" was automatically created in Apsis. Profiles matching the query were tagged with this label, making them easily filterable in segments and audience views.

### Timing and Results

- Import status shows "In calculating total" initially
- Processing may take a minute or two
- Once complete, profiles can be searched by tag in **Audience** or filtered in **Segments**

[Lukasz Grabowski]: Opened one of the tagged profiles (CRM ID 87) and confirmed it had the "FS Enterprise" tag applied.

---

## Data Consistency and Synchronization Challenges

### Attribute Mapping Gaps and Validation

[Michal Rosikiewicz]: Noticed that a "mobile" field was mapped but did not appear in synced profiles.

[Shreevidhya Ganesan]: Explained that field validation logic might skip fields if validation fails. If a phone number format is invalid, the entire field might be omitted from the sync.

Evidence in logs (Delta Sync Worker):
```
Updating attribute 'mobile' with delete status — failed validation
Tried to validate and format phone number
```

This illustrates that **strict validation is applied** and invalid data is excluded rather than corrupting the record.

### Profile Identification

Two distinct IDs identify a profile across systems:

- **CRM ID**: The profile's ID in the external system (e.g., 179 in FEC)
- **Profile Key**: The internal Apsis identifier generated upon import
- **Integration Key**: Used to trace logs and operations

[Lukasz Grabowski]: Observed that when searching for profile 179 in Apsis Audience, it appeared with a different internal profile key, but the CRM ID was visible in the profile link.

---

## Multi-Section and Multi-Account Integration Limitations

### Per-Section Limitation

[Michal Rosikiewicz]: Asked if multiple integrations can exist in one account, and how many accounts can use the same CRM integration.

[Shreevidhya Ganesan]: Clarified the constraint:

- **Efficy Enterprise**: Can have multiple installations across different sections in the same account (no limit mentioned)
- **Microsoft Dynamics** and **Shopify**: Maximum of **one installation per section**—one instance can only be installed in one section

Reason: A feature called **deep linking** is unique to Dynamics and Shopify. Deep linking allows users to select a profile and navigate directly to that profile in the external system. This requires a single, unified connection.

### Testing with Multiple Environments

For development and testing purposes:

- Each CRM has a shared **test instance** with credentials stored in **LastPass**
- Developers can copy the API key and URL from LastPass and paste them into new sections for testing
- This allows multiple developers to test independently in the same test environment (unlike production, where only one integration per section is allowed for certain systems)

[Tomasz Kowalski]: Asked how to set up new accounts for testing.

[Shreevidhya Ganesan]: Explained the LastPass approach: *"For most of the integrations we have the API key and URL in the LastPass. So if you try to search with the name like FEC corporate or tribe, you can try tribe. Let me see if we have it... you have tribe."*

For systems without a front-end UI (no public interface), testing is done once per integration type on a representative environment, since the generic connector's code is shared across all instances.

---

## Full Sync Mechanics and Workflow

### When Full Sync Is Required

1. **Initial setup**: Required once after installation to import all existing profiles from the external system
2. **Data realignment**: After major changes or when full historical data needs to be re-imported
3. **Sync condition testing**: To verify sync conditions work correctly on bulk profile sets

[Shreevidhya Ganesan]: Emphasized that once delta sync is running, full sync is not needed for ongoing changes. Full sync's primary purpose is to synchronize existing data that predates the integration.

### Full Sync with Sync Conditions

Full sync respects sync conditions. The team demonstrated:

1. Created a sync condition: `Email equals [lucas.gr@...]`
2. Triggered full sync
3. Result: 1 profile synced (matching the condition), 89 profiles skipped (not matching)

This proves sync conditions are applied before import, not after.

### Full Sync Report

After full sync completes, a **Full Sync Report** is available showing:
- Total profiles processed
- Successfully synced profiles
- Skipped profiles
- Sync status (in progress, completed, etc.)

---

## Architecture Insights and Future Learning

### Generic Connector Approach

[Lukasz Grabowski]: Asked about the process for onboarding a completely new external system.

[Shreevidhya Ganesan]: Explained that the shift to a generic connector means:
- Standardized specifications are defined
- External systems must comply with the specification
- Once compliant, setup becomes routine
- Postman collection available for testing all endpoints

> "Since it's like a generic connector testing, one installation is fine for us to test the feature and all the installation is going. All the connector is going to share the same code of the service, right?"

This means testing can be done on one environment; all instances share the same backend code.

### Recommended Learning Path

[Shreevidhya Ganesan]: Suggested that understanding the architecture first will build confidence:

> "Once you know the architecture, you will become more confident with the whole workflow and after that you will feel yourself more confident in the position that you can handle it by yourself because the generic specification whatever we have developed is like more standardized and it's like straightforward."

Deep dives into specific areas (webhook registration, profile queries, advanced filtering) are covered in follow-up sessions with **Eric**.

---

## Key Takeaways

1. **Generic Connector is the standard approach**: All new integrations use the generic connector architecture, which is more standardized and performant than legacy systems.

2. **Delta Sync is continuous and webhook-driven**: Once installed, delta sync automatically monitors changes via webhook callbacks. Latency varies by external system (immediate for Dynamics, minutes for Efficy Enterprise).

3. **Full Sync brings historical data into alignment**: Separate from delta sync, full sync imports all existing profiles from the external system. It's required once per integration setup, then delta sync maintains synchronization.

4. **Attributes sync unidirectionally, consent is bidirectional**:
   - Changes to mapped attributes in Apsis do NOT sync back to the external CRM
   - Consent changes (opt-in/opt-out) sync in both directions automatically
   - This prevents data conflicts while keeping consent state synchronized

5. **Sync conditions are Apsis-side filters**: Multiple conditions use AND logic; all must be satisfied for a profile to sync. Filtering happens in Apsis, not in the external CRM.

6. **Queries enable automatic profile tagging**: Predefined queries in the external system can be imported to automatically tag matching profiles in Apsis, enabling segmentation.

7. **CloudWatch logs enable detailed debugging**:
   - Delta Sync Manager logs show webhook payloads
   - Delta Sync Worker logs show validation errors and processing details
   - Use integration key format (`account_id/section_id/integration_name`) to filter logs

8. **Multi-section integrations vary by system**:
   - Efficy Enterprise: Multiple installations per account (across sections)
   - Dynamics/Shopify: One installation per section (due to deep linking feature)

9. **Testing uses shared test environments**: API keys and URLs from LastPass allow developers to independently test on shared test instances without affecting other developers.

10. **Postman collection available for endpoint testing**: Developers can test the generic connector specification endpoints independently, needing only API key and CRM URL.

---

## Unresolved Questions and Follow-Up Items

1. **API key creation in FEC 12.1**: How to create new integration credentials—escalated to enterprise team for clarification.

2. **Phone number validation rules**: The exact validation logic for phone numbers that caused the "mobile" field to be skipped was not detailed; Eric will cover this in architecture deep dive.

3. **Query creation in Efficy**: The team used existing queries but did not create new ones. [Shreevidhya Ganesan] noted this is complex and deferred to Eric for detailed explanation.

4. **Webhook registration details**: The exact payload and registration mechanism was mentioned but not fully explored. Eric will deep dive into this in the next session.

5. **Field mapping from external system perspective**: Why some fields (e.g., first name/last name) appeared as a single "name" field in the external system profiles—may be a language/localization issue (French interface observed).

6. **Recurring query imports**: The team set up a query with "Recurring" option (e.g., daily), but the exact cron schedule and behavior were not fully explored.

7. **Onboarding process for new external systems**: The high-level process was explained, but a full workshop with a hypothetical new external system was suggested as future learning.

---

**Session conducted by**: Shreevidhya Ganesan (domain expert)  
**Participants**: Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski  
**Duration**: 57 minutes 3 seconds  
**Follow-up**: Deep dive with Eric on webhook registration, query creation, and advanced logging expected.