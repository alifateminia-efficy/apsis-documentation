---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Installation and Configuration, Field Mappings, Delta Sync vs Full Sync, Subscription Mapping, Sync Conditions, Queries and Tags, Bidirectional Consent Sync, Logging and Debugging, Generic Connector Architecture]
speakers: [Shreevidhya Ganesan, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Efficy Enterprise 12.1, Generic Connector, Delta Sync Manager, Delta Sync Worker, Cloudwatch Logs, Field Mapping, Subscription Mapping, Sync Conditions, Queries and Profiles, Webhook Callbacks]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Inbound Flow, Outbound Flow, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session covered the setup and operation of the Efficy Enterprise 12.1 connector using the generic connector architecture. The team walked through a practical installation, demonstrating field mappings, real-time delta synchronization, subscription mapping, sync conditions, and query-based profile tagging. Key concepts covered include the distinction between delta sync (real-time incremental changes) and full sync (bulk historical data synchronization), bidirectional consent synchronization, and how to navigate Cloudwatch logs for debugging. The session emphasized that the generic connector is a standardized approach that external systems must adhere to, making integration more straightforward than legacy solutions.

---

## Installation and Configuration of Efficy Enterprise 12.1

### Setting Up the Integration

The Efficy Enterprise 12.1 integration requires two key URLs: one for the external CRM system and one for the Apsis One installation environment.

**Important Limitation**: You can only have one installation per section in Apsis One. [Shreevidhya Ganesan]: If you want to test multiple versions or instances of the same external system (e.g., FEC 12.0 and FEC 12.1), you need to create separate sections within the same account.

> "You can have like more sections because like you know you can have like one CRM installed in every section in case if you want to explore." [Shreevidhya Ganesan]

It is recommended to name the integration descriptively (e.g., "EE 12.1") to make it easy to identify which installation version you have in a given section.

### API Key and Credentials

The API key and environment details are typically provided by the external CRM system itself. The team did not have the ability to create new API keys or integrations directly in the FEC 12.1 CRM during this session—these permissions are managed by the CRM administrator. For testing purposes, the team uses shared test environments stored in LastPass with API credentials for systems like Tribe and FEC Corporate.

### Field Mapping Setup

After installation, configure field mappings between Apsis One profiles and the external system attributes. The generic connector supports an **auto-mapping** feature that can automatically match common fields.

[Shreevidhya Ganesan]: "If you want you can do auto mappings and then save."

When creating a test profile, you can map standard fields like:
- Name
- Email
- Phone Number
- Mobile
- Language
- Country
- Banner

**Validation note**: Phone numbers must adhere to the system's format requirements. For example, a phone number like `+46700...` should follow the correct format to pass validation.

---

## Delta Sync vs Full Sync: Understanding the Two Sync Mechanisms

### Delta Sync (Real-Time Incremental Synchronization)

Delta sync is the **default, real-time synchronization mechanism** that starts automatically once the integration is installed. It works by:

1. **Registration of Webhook Callbacks**: During installation, Apsis One registers a callback URL with the external CRM system.
2. **Webhook Triggering**: When a profile changes in the external CRM, the system triggers the callback URL with the profile ID and changed field information.
3. **Field-Level Filtering**: Only the fields that have been mapped in Apsis One are tracked for changes. The webhook payload includes which fields have been mapped so the external system knows which changes to report.

[Shreevidhya Ganesan]: "When we do a registration we have, we will be sending the fields whatever we have mapped in our system to the external system. So the so that the external system will be knowing that these are the fields which access has been mapped for. And any changes in that field, the web hook will be triggered."

**Processing Delay**: For Efficy Enterprise systems, delta sync changes are processed in batches rather than immediately:

[Shreevidhya Ganesan]: "For FSC enterprise it takes like you know couple of minutes for batch processing to happen like you know they try to collect the changes and try to send it in short I guess."

In contrast, Microsoft Dynamics triggers delta sync almost immediately.

### Full Sync (Bulk Historical Synchronization)

Full sync is necessary to bring existing data into sync between the external system and Apsis One. It should be used:

- **First time after installation**: To pull all existing records from the external system
- **When historical data needs synchronization**: To synchronize profiles that haven't changed recently and would not have been captured by delta sync

[Shreevidhya Ganesan]: "Full sync maybe like you know it's required for the first time because you want to pull all the records from the external system, right? ... When you do like start with the full sync you can bring the access profiles and the external system profiles on the same ground. And then maybe like you know from there onwards the Delta Sync will try to keep the contacts in CRM and apps in sync."

**Important caveat**: Delta sync runs continuously regardless of whether a full sync has been done. A full sync is supplementary, not prerequisite.

---

## Subscription Mapping and Consent Management

### Creating Subscriptions

Subscriptions in Apsis One act as profile lists that can be mapped to consent states in the external CRM system.

**Workflow**:
1. Create a folder to organize subscriptions (e.g., "Enterprise 12.1")
2. Create subscriptions within that folder (e.g., "Profile QA")
3. Map subscriptions to external system profiles in the integration settings

[Shreevidhya Ganesan]: "If you go here I can create folder which is enterprise... And then you can use it here in mapping."

### Bidirectional Consent Synchronization

**Critical distinction**: While **attributes are unidirectional** (external system → Apsis One only), **consent states are bidirectional**.

**Inbound (External System → Apsis One)**:
- When a profile is added to a subscription in the external system, delta sync updates the consent state in Apsis One.

**Outbound (Apsis One → External System)**:
- When a user changes their opt-in/opt-out status in Apsis One, this change is synchronized back to the external system.

**Consent States**:
- No subscription (initial state)
- Opted in
- Opted out

[Shreevidhya Ganesan]: "Only the consent, yeah here, not the attributes is not bidirectional. Consent is bidirectional."

The outbound consent changes are processed through the outbound sync mechanism and may take a few minutes to be reflected in the external system.

---

## Understanding and Navigating Logs for Debugging

### Log Architecture

The integration uses a two-stage logging system in Cloudwatch:

1. **Delta Sync Manager**: Produces and formats messages from webhook callbacks
2. **Delta Sync Worker**: Processes the formatted messages and applies validation and transformation logic

[Shreevidhya Ganesan]: "Delta Sync Manager is where you just produce the messages and then the processing will happen in go to log groups and you have Delta Sync worker. This is only up to the part where you produce the messages in the format what our worker can start processing with."

### Log Entry Structure and Integration Key

Each log entry contains an **integration key** that uniquely identifies the data source:

```
integration_key: {account_id}/{section_id}/{integration_name}
message_group_id: {provides additional context}
entry_id: {unique message identifier}
context: {CRM ID of the profile being synced}
```

**How to use the integration key**: When searching Cloudwatch logs, use the integration key to filter for all messages related to a specific integration. For example:

```
account_id/section_id/Enterprise12.1
```

### Common Log Patterns and Error Resolution

**Phone Number Validation Failure**:
When a phone number field fails validation (e.g., too short or incorrect format), the Delta Sync Worker logs a validation error:

```
"tried to validate and format phone number. That's that's the reason."
```

This occurs during attribute mapping. If validation fails, the field is skipped and not synced.

[Shreevidhya Ganesan]: "If the validation doesn't happen correctly then we skip."

**Navigating to Relevant Logs**:
1. Go to Cloudwatch log groups
2. Select "Delta Sync Manager" for webhook message production logs
3. Select "Delta Sync Worker" for processing and validation errors
4. Filter by the integration key (account/section/integration) to find messages for your specific integration
5. Look within the past 5-10 minutes depending on when the sync occurred

[Michal Rosikiewicz]: "Can you show us around the logs where we can find those errors or our information that something is in sync right now?"

---

## Sync Conditions: Filtering Which Profiles Get Synchronized

### Purpose and Behavior

Sync conditions allow you to filter which profiles from the external system are imported into Apsis One. All sync conditions must be satisfied (AND logic) for a profile to be synced.

[Shreevidhya Ganesan]: "All the sync conditions should be satisfied."

### Creating Sync Conditions

1. Navigate to the integration settings
2. Add a new sync condition rule
3. Select an attribute from the external system
4. Choose a comparison operator
5. Provide a value

**Example**: Create a condition where `Email = {specific_email_address}`. Only profiles matching this email will be synced during full sync or delta sync.

### How Sync Conditions Apply

**Both to real-time and batch operations**: 
- When delta sync receives an update notification, it checks against sync conditions before importing
- When full sync runs, profiles are evaluated against conditions and non-matching profiles are skipped

[Shreevidhya Ganesan]: "Say you have the sync condition for this and you're going to get one update message which is not satisfying the sync condition, then it will be through... it's not for full sync or any any full sync or real time sync. The sync condition should match."

**Filtering happens on Apsis side, not the external system side**:

[Michal Rosikiewicz]: "This is filtered on our site or we say to CRM?"

[Shreevidhya Ganesan]: "No, in our side, in our side, it's not with the CM."

### Multiple Conditions (AND Logic)

If you add multiple conditions, all must be true for a profile to sync:

```
Condition 1: Email = user@fec.com
Condition 2: Name = Mikhail
```

Only profiles where BOTH conditions are true will be imported.

[Shreevidhya Ganesan]: "Say now when you do this, only profiles satisfying this condition, only only the one profile will be synced because that's the only profile which will be matching the condition of the e-mail, right?"

### Full Sync Report

After a full sync, the Full Sync Report shows:
- Number of profiles successfully synced
- Number of profiles skipped (due to sync conditions not being met)
- Status of the sync operation

---

## Queries and Profile Tagging

### What Queries Do

Queries are saved searches or segments within the external CRM system that identify groups of profiles. When you import profiles from a query, those profiles are **tagged** in Apsis One with the query name.

[Shreevidhya Ganesan]: "It's like a kind of you can how how to imagine that is say for example this feature comes in handy when you have you want to tag profile say there will be like a query in the external system where you can say the profiles which which are in VIP status or something."

**Practical use case**: A query in the external CRM identifies all "VIP" customers. When you import these profiles using the query, they get tagged with "VIP" in Apsis One, making them easy to filter in segmentation and campaign targeting.

### Importing Profiles from a Query

1. Navigate to **Queries and Profiles** in the integration
2. Select an existing query from the external system
3. Choose **Start Import** to immediately import profiles matching that query
4. Optionally, enable **Recurring** to have the import run on a schedule (e.g., daily)

[Shreevidhya Ganesan]: "Now you can just see like you know you can just start the import immediately. One is start import."

### Recurring vs One-Time Imports

- **One-time import**: Only runs once when you click "Start Import"
- **Recurring import**: If checked, a cron job runs daily to keep the profile list in sync with the external system

[Shreevidhya Ganesan]: "Recurring is like every day. It's if you do not check recurring, it just happens one. If you check recurring like every day morning, there's a cron job which will keep this profile list In Sync with our system."

**Note**: Queries and profile tagging is separate from full sync. Full sync imports all profiles matching your sync conditions; query imports tag a specific subset of profiles.

---

## Attribute Synchronization: Directional Flow and Limitations

### Unidirectional Attribute Sync (External System → Apsis One)

**Attributes only flow from the external CRM into Apsis One**, not the other way around.

[Shreevidhya Ganesan]: "The attributes are not sent back, but but there is a curve no... Whatever the fields you have mapped, only those fields will be reflected."

**Implication**: If you edit a profile attribute in Apsis One, that change will NOT be reflected in the external CRM. However, if you edit the same attribute in the external CRM, it will overwrite the Apsis One value when synced.

> "If I change here something and then go and change here, so we will overwrite by it will be overwritten by data from here, this problem, right?" [Lukasz Grabowski]

[Shreevidhya Ganesan]: "Exactly, exactly."

### Which Fields Are Synced

Only fields that have been **mapped in the field mapping configuration** are synced. Unmapped fields are ignored.

---

## Generic Connector Architecture and Standardization

### Shift to Generic Connector Approach

The platform has moved away from building custom connectors for each external system toward a **generic connector** that any external system can use if they meet the standard specification.

[Shreevidhya Ganesan]: "We just moved to the approach of having the generic connector approach. So any of the external system which wants to get connected with our integrations will be having like a set of standard specification which they have to adhere to, yeah."

### Standardized Specification

New external systems must adhere to a standardized API specification that defines:
- Webhook registration and callback format
- Field mapping payload structure
- Consent state representation
- Query/segment export format

[Shreevidhya Ganesan]: "Once if you know the architecture, you will become more confident with the the whole workflow and after that you will feel yourself like more confident in the position that you can handle it by yourself because the generic specification whatever we have developed is like more standardized and it's like a straightforward."

### Testing New Integrations

To test a new integration, you need:
- The external CRM's API key and base URL
- Access to a test instance of that CRM
- Postman collection for testing endpoints directly

[Shreevidhya Ganesan]: "For you to try it out there is like a postman collection also what we have. So you can try all those endpoints by yourself like making a a simple postman request which needs only the API key and the prefixing URL of the external CRM right?"

---

## Limitations by Connector Type

### Efficy Enterprise 12.1 (FEC)

- **Supports multiple installations per account**: You can have different FEC instances (12.0, 12.1, etc.) installed in different sections within the same account
- **Batch processing delay**: Delta sync uses batch processing, taking a few minutes to propagate changes

### Microsoft Dynamics and Tribe

- **One installation per account maximum**: These systems do not support multiple concurrent installations within the same account
- **Reason**: Deep linking feature requires a single canonical link to profiles in the external system

[Shreevidhya Ganesan]: "When it comes to Maxo or Dynamics side shop, you cannot have it on the multiple sections. And the there's a reason behind it. Maybe like you will get to know as you deep dive into it because there's a feature called like you know deep linking where you can just select the profile and it directly takes you to the profile in the external system."

---

## Shared Test Instances and Setup for Multiple Accounts

### Accessing Test Environments

For each external CRM system, there is typically a shared test environment with credentials stored in LastPass:

[Shreevidhya Ganesan]: "Every every CRM will have its own test instance and they have shared the test instance API key and URL with that that we have put into the last pass."

**For FEC systems**: Search LastPass for the CRM name (e.g., "FEC Corporate", "Tribe") and retrieve the API key and URL.

**Setup workflow**:
1. Copy API key and URL from LastPass for the test CRM
2. Go to a new section in your Apsis One account
3. Create a new integration installation
4. Paste the API key and URL
5. Complete the configuration (field mappings, subscription mappings, etc.)

### Limitations on Front-End Access

Some external CRM systems do not provide a front-end UI for testing, only API access. In these cases, the team has not tested the full UI flow but relies on API testing.

[Shreevidhya Ganesan]: "For couple of environments we do not have the front end link for the CRM system so I myself have not tested it's like you know since it's like a generic connector testing, one installation is fine for us to test the feature and all the installation is going. All the connector is going to share the same code of the service, right?"

---

## Key Takeaways

1. **Delta sync is real-time and automatic**: It starts immediately upon installation and continuously synchronizes changes from the external CRM without manual intervention.

2. **Full sync is for historical data**: Use it to bring existing records into sync, especially after installation or when you need a complete data refresh.

3. **Attributes are unidirectional**: Changes made in Apsis One are not reflected back to the external system. Always consider the external system as the source of truth for attribute data.

4. **Consent is bidirectional**: Opt-in/opt-out status synchronizes in both directions, making it safe to manage consent in either system.

5. **Sync conditions filter at import time**: They are evaluated on Apsis One's side and apply to both real-time and batch syncs. All conditions must be satisfied (AND logic) for a profile to import.

6. **Queries enable profile tagging**: Use queries from the external system to tag subsets of profiles in Apsis One for easier segmentation and campaign targeting.

7. **Logs are the primary debugging tool**: Use Cloudwatch logs with the integration key (account/section/integration) to track sync activity and identify validation errors.

8. **Generic connector is the standard approach**: New external systems must adhere to a standardized specification. Once you understand the architecture, integrating new systems becomes straightforward.

9. **Different connectors have different limitations**: FEC supports multiple installations per account; Dynamics and Tribe do not (due to deep linking constraints).

10. **Phone number validation is strict**: Phone fields must conform to expected formats, or they will be skipped during sync.

---

## Unresolved Questions and Action Items

1. **How to create API keys in FEC 12.1 CRM**: This requires permissions that the team does not have. Should be explored with the enterprise/CRM team if needed for production setups.

2. **Why is mobile attribute missing in some syncs?**: The team observed that mapped mobile fields were sometimes skipped. This may be due to validation failures on incomplete phone numbers. Eric was scheduled to provide deeper insight into validation logic.

3. **Understanding the query creation process**: The team did not attempt to create a custom query from scratch—only imported from existing queries in the external system. This should be covered in a future session with Eric's guidance.

4. **Outbound consent sync completion**: During the session, the team could not confirm that an opt-in change made in the external system was reflected back in Apsis One in real time, though this is expected behavior. Should be verified once processing completes.

5. **Workshop on onboarding new external systems**: Lukasz expressed interest in a future workshop covering the process of onboarding a completely new external system (beyond FEC, Dynamics, or Tribe). This could involve using the Postman collection to test endpoints directly.