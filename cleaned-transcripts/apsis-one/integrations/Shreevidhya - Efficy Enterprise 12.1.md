---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One - Integrations
topics: [FEC Enterprise 12.1 Installation, Generic Connector Setup, Field Mappings, Delta Sync, Full Sync, Sync Conditions, Queries and Profiles, Consent Management, Bidirectional Sync, API Integration, Logging and Debugging]
speakers: [Shreevidhya Ganesan, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Delta Sync Manager, Delta Sync Worker, Cloudwatch Logs, Webhook Callbacks, Field Mapping, Subscription Mapping, Sync Conditions, Full Sync, Queries and Profiles, Consent Timeline]
session_type: knowledge-transfer
---

## Session Overview

This session walked through a hands-on setup and testing of **Efficy Enterprise (FEC) 12.1** using the **generic connector** integration approach in Apsis One. The team demonstrated installation in a new section, field mapping configuration, delta sync mechanics with real-time webhook callbacks, consent synchronization, sync conditions for filtering profiles, and the queries/profiles feature for tagging. Key architectural concepts were explored including the distinction between unidirectional attribute sync (CRM → Apsis) and bidirectional consent sync, and the debug logging strategy via CloudWatch.

---

## Installation and Environment Setup

### Section-Based Installation Architecture

[Shreevidhya Ganesan]: The integration requires two URLs: one for the CRM and one for the Apsis installation. The key limitation discussed is that **only one integration instance per section** is allowed.

[Lukasz Grabowski]: When setting up a new section (e.g., "enterprise integrations"), you can have multiple sections within a single account, allowing exploration of different CRM installations. This is useful for testing purposes.

[Shreevidhya Ganesan]: You can name the section integration link descriptively (e.g., "EE 12.1") to easily identify which installation you're working with, though the name is a reference link and cannot be directly edited—it continues to function correctly.

### API Key and Environment Configuration

[Michal Rosikiewicz]: Asked how to create API keys in FEC 12.1. 

[Shreevidhya Ganesan]: The environment and API key were provided by the CRM test instance. The team does not have permissions to create new integrations; they use pre-provisioned credentials. These credentials are typically stored in LastPass and can be discovered by searching for the CRM name (e.g., "FEC Enterprise" or "Tribe").

---

## Field Mapping and Record Creation

### Auto-Mapping and Manual Configuration

[Shreevidhya Ganesan]: After installation, field mappings can be configured. The system supports **auto-mapping** functionality, which can be triggered and saved to automatically align fields between the CRM and Apsis.

[Lukasz Grabowski]: When creating a new contact record for testing, mapped fields appear (e.g., name, country, email, title, language, banner). Unmapped fields (like Skype) are skipped.

**Important caveat on phone number validation**: Phone numbers must follow a valid format (e.g., `+46700xxx`). If validation fails, the field is skipped during sync processing.

### Direct Record Creation vs. Full Sync

[Shreevidhya Ganesan]: Rather than starting with a full sync, the team created a test record directly in the external system to verify the integration mechanics. This is faster for initial validation than waiting for bulk sync operations.

Record creation in the generic connector is faster than in legacy systems because it's a purpose-built, specialized implementation.

---

## Delta Sync Mechanics and Real-Time Synchronization

### How Delta Sync is Triggered

[Shreevidhya Ganesan]: Delta sync (real-time synchronization) begins automatically after installation. The mechanism works as follows:

1. **Webhook Registration**: During installation, the integration registers a callback URL with the external CRM.
2. **Change Notification**: When a change occurs in the external CRM, the CRM notifies Apsis by triggering the callback URL with the profile ID that changed.
3. **Processed Fields**: Only fields that have been mapped are included in the webhook payload. The external system knows which fields to monitor based on the mapping configuration sent during registration.

[Tomasz Kowalski]: Asked what specifically triggers a delta sync—whether it's all changes or only mapped attribute changes.

[Shreevidhya Ganesan]: The registration includes the mapped fields in the payload. Any changes to those mapped fields trigger the webhook. This is part of the contract between Apsis and the external CRM.

### Full Sync vs. Delta Sync

[Shreevidhya Ganesan]: **Delta sync alone is insufficient for initial data alignment**. Full sync is required when:
- You want to pull all existing records from the external system for the first time.
- You need to bring existing data (where no recent changes occurred) into synchronization.

After a full sync establishes initial parity, delta sync keeps the systems in ongoing sync for new changes and newly created records.

---

## Logging and Debugging Strategy

### CloudWatch Log Navigation

[Michal Rosikiewicz]: Requested information on where to find sync logs and errors.

[Shreevidhya Ganesan]: Logs are accessible via CloudWatch in the stage environment. There are two key log groups:

1. **Delta Sync Manager** (`log_groups` → `Delta Sync Manager`): Produces webhook messages in the format required by the worker.
2. **Delta Sync Worker** (`log_groups` → `Delta Sync Worker`): Processes the messages and applies business logic (validation, transformation, etc.).

### Integration Key Structure for Log Searching

The **integration key** is the primary identifier for searching logs and contains the following hierarchy:

```
{account_id}/{section_id}/{integration_id}
```

Within each log entry:
- **Message Group ID**: Contains metadata about the CRM ID and profile information
- **Entry ID**: Further distinguishes individual sync operations
- **Context**: Shows attribute changes (e.g., "updating attribute mobile with delete status")

[Lukasz Grabowski]: Once you construct the integration key (account/section/integration), you can filter CloudWatch logs to see all sync activity for that specific integration.

### Example: Phone Number Validation Failure

[Michal Rosikiewicz]: Observed that the "mobile" field created during field mapping did not appear in the synced profile.

[Shreevidhya Ganesan]: The Delta Sync Worker logs showed a validation error: "tried to validate and format phone number [value] — phone number is too short". The field was skipped because validation failed.

This demonstrates that data validation happens in the worker, and invalid data is excluded from sync rather than causing the entire operation to fail.

---

## Consent and Subscription Management

### Bidirectional Consent Synchronization

[Shreevidhya Ganesan]: **Consent is bidirectional; attributes are unidirectional (CRM → Apsis only)**.

- **Attributes**: Changes flow only from the external CRM to Apsis. Editing attributes in Apsis does not push back to the CRM.
- **Consent**: Changes flow in both directions. Opt-in/opt-out state changes sync between Apsis and the external CRM.

[Lukasz Grabowski]: Tested this by creating a subscription mapping linking profile "QA" to the integration, then testing consent changes.

### Subscription Mapping and Profile Linking

To enable consent sync, subscriptions must be:
1. **Created in the Apsis section** (e.g., a folder called "Enterprise 12.1").
2. **Mapped to the external CRM profile** in the subscription mapping configuration (e.g., mapping profile "QA" to email subscription).

Once mapped, profile consent changes in either system trigger delta sync updates.

### Testing Consent Changes

[Shreevidhya Ganesan & Lukasz Grabowski]: Demonstrated the workflow:

1. Created a profile in the external CRM (Lucas GR).
2. Added the profile to a mapped profile/subscription (Profile QA).
3. This triggered a delta sync that created a corresponding profile in Apsis (ID: 179).
4. In Apsis, changed the consent state from "opted in" to "opted out" (via subscription unsubscribe).
5. Verified in the external system that the profile state changed to "opted out" within minutes (batch processing delay).
6. Changed consent back to "opted in" in the external CRM; Apsis reflected this after another sync cycle.

**Processing Delay**: FEC Enterprise uses batch processing for outbound consent changes. Unlike Dynamics, which updates immediately, FEC collects changes and sends them in batches, resulting in a delay of a couple of minutes.

---

## Sync Conditions: Filtering Which Profiles to Sync

### Concept and Use Case

[Shreevidhya Ganesan]: Sync conditions allow you to filter which profiles are synchronized based on attribute values. They work as **AND conditions**—all conditions must be satisfied for a profile to sync.

> "A profile to be synced, all the conditions should satisfy."

### Creating and Applying Sync Conditions

[Shreevidhya Ganesan]: The team added a sync condition:
- **Attribute**: Email
- **Operator**: Equals
- **Value**: `lukasz.grabowski@...`

This restriction means only profiles matching this email address would be synced.

**Note on "Contains" Operator**: The `contains` operator is a newer feature that is currently behind a feature flag and requires external CRM support to function. The basic operators (equals, etc.) are universally supported.

### Client-Side Filtering (Not Server-Side)

[Michal Rosikiewicz]: Asked whether sync conditions are enforced on the external CRM side or in Apsis.

[Shreevidhya Ganesan]: Conditions are evaluated **in Apsis, not in the external CRM**. The external system sends all profiles, and Apsis filters them client-side. This is why the full sync report showed "90 profiles retrieved, 1 matched, 89 skipped."

### Test Results

When a full sync was triggered with the condition in place:
- **Total profiles in external system**: 90
- **Profiles matching condition**: 1 (only the specified email)
- **Profiles skipped**: 89 (did not match the condition)
- **Result**: Only 1 profile successfully synced to Apsis

When the sync condition was removed:
- All profiles were eligible for sync again.

### Scenario: Multiple Conditions (AND Logic)

[Shreevidhya Ganesan]: If you add a second condition (e.g., `name = "Mikhail"`), a profile must satisfy **both** the email condition **and** the name condition to sync. If you edit the name to something other than "Mikhail", the profile would fail the second condition and be skipped, even if it previously synced.

---

## Queries and Profiles: Tagging for Segmentation

### What Queries and Profiles Do

[Shreevidhya Ganesan]: Queries in the external CRM system are like saved segments or groups. The "Queries and Profiles" feature imports profiles from an external system query and automatically **tags them** in Apsis.

**Use case example**: 
> "Say there will be like a query in the external system where you can say the profiles which are in VIP status or something. So when you try to import those profiles and have the profiles imported, you will be having those profile tagged as VIP so that in segmentation it can become handy to just filter out the profiles which has the tag VIP."

[Lukasz Grabowski]: The tag name becomes the integration name (e.g., "FEC Enterprise") and can be used to create segments in Apsis based on profiles imported from that query.

### Manual Import vs. Recurring Import

[Shreevidhya Ganesan]: The "Recurring" checkbox controls how frequently the import runs:
- **Unchecked**: One-time import only.
- **Checked**: Runs as a recurring cron job (e.g., daily) to keep the profile list in sync with the external system.

### Import Process

[Lukasz Grabowski]: Selected the query "apsis_test" and initiated a manual import via **Start Import**.

The system then:
1. **Calculated total profiles** matching the query (status: "In calculating total").
2. **Processed the profiles** (5 profiles were successfully processed).
3. **Tagged the profiles** with the corresponding tag in Apsis.

### Challenges with Queries and Profiles

[Shreevidhya Ganesan]: Creating custom queries in the external CRM system is complex. For all testing, the team has used pre-existing queries in the external system rather than building new ones. Eric (a team member) may provide more guidance on query creation in a follow-up session.

### Verification: Imported Profiles

After the import completed, profiles appeared in Apsis with:
- Limited field data (e.g., email only, without first/last name in this case).
- A **tag** identifying them as part of the imported query (tag: "FSC Enterprise").

The tag enables later segmentation: you can create segments that include only profiles with the "FSC Enterprise" tag.

---

## Field Mapping Directionality and Data Overwrite Behavior

### Attributes Are Unidirectional Only

[Shreevidhya Ganesan]: Mapped attributes flow only from the external CRM to Apsis. If you edit an attribute in Apsis and separately edit it in the external CRM, the external CRM value will overwrite the Apsis value on the next sync.

[Lukasz Grabowski]: This is an important caveat for data integrity:

> "If I change here something and then go and change here, so we will overwrite by it will be overwritten by data from here."

**Only mapped fields are synced**. If you map "middle_name" in the field mapping but not "phone", then "phone" changes in Apsis are safe from overwrite, but "middle_name" changes will be overwritten by the next CRM sync.

---

## Account and Instance Scalability Limitations

### FEC Enterprise: Multiple Installations Per Account

[Michal Rosikiewicz]: Asked if the same integration can be installed on multiple accounts.

[Shreevidhya Ganesan]: Yes, for **FEC Enterprise**, the same integration can be installed in multiple sections of a single account, each with its own configuration and API key. However, only one instance per section.

### Dynamics and Other Systems: Single Installation Restriction

[Shreevidhya Ganesan]: For **Dynamics CRM** and **Shopify**, only one integration instance can be installed per section. You cannot have multiple installations of the same CRM in different sections of the same account.

**Reason**: The **deep linking** feature. Deep linking allows you to click a profile in Apsis and be taken directly to that profile in the external CRM. This one-to-one relationship between Apsis and the external system prevents multiple installations.

---

## Setting Up Test Environments for Development and Testing

### Obtaining Test Credentials

[Tomasz Kowalski]: Asked how to set up multiple accounts for testing and development when restricted to one integration per section.

[Shreevidhya Ganesan]: Each external CRM vendor provides a shared test instance with API credentials stored in **LastPass**. To set up a new account:

1. Search LastPass for the CRM name (e.g., "Tribe", "FEC Corporate").
2. Copy the API key and base URL from the LastPass entry.
3. Paste the credentials into a new section during installation.
4. The installation will connect to that vendor's shared test environment.

### Limited Front-End Access

[Shreevidhya Ganesan]: For some CRM systems, there is no front-end UI provided for testing, only API access. In these cases, testing is limited to the generic connector code path (which is sufficient since all connectors share the same underlying service code).

---

## Generic Connector: Standardized Specification Approach

### Shift from Legacy to Generic Connector

[Shreevidhya Ganesan]: The team has moved away from building specialized connectors for each CRM and adopted a **generic connector approach**. Any external CRM system that wants to integrate with Apsis must adhere to a standard specification.

[Lukasz Grabowski]: Asked if there's a process for onboarding new external systems.

[Shreevidhya Ganesan]: Yes. Any new external CRM is required to:
1. Implement the standard **generic connector specification**.
2. Provide API endpoints that conform to the specification.
3. Support webhook callbacks for delta sync.

This approach is much more scalable than building a unique connector for each system.

### Postman Collection for Testing

[Shreevidhya Ganesan]: A **Postman collection** is available to test the generic connector specification. It includes all standard endpoints and requires only:
- API key
- Base URL prefix of the external CRM

Teams can use Postman to validate that a new CRM's API conforms to the specification before full integration work begins.

---

## Future Workshop Opportunity: Adding a New External System

[Lukasz Grabowski]: Proposed a future workshop scenario: "We have a new external system [request to add]. What do we do?"

[Shreevidhya Ganesan]: This is definitely possible. The workshop would likely cover:
1. Understanding the generic connector architecture.
2. Reviewing the standard specification.
3. Validating the external CRM's API against the spec using Postman.
4. Configuring the integration in Apsis (installation, field mapping, etc.).

The standardized approach makes the process straightforward and reduces the learning curve once the architecture is understood.

---

## Key Takeaways

1. **Installation Model**: FEC Enterprise allows multiple installations per account (different sections); Dynamics and Shopify do not (one per section max, due to deep linking).

2. **Delta Sync Mechanism**: Webhook callbacks are registered during installation for mapped fields only. The external CRM notifies Apsis of changes in real-time, with batch processing delays on FEC (couple of minutes).

3. **Data Flow Direction**:
   - **Attributes**: Unidirectional (CRM → Apsis only). Changes in Apsis are overwritten by the next CRM sync.
   - **Consent**: Bidirectional. Opt-in/opt-out changes sync both ways.

4. **Sync Conditions**: Client-side filtering applied in Apsis (not the external system). All conditions must be satisfied (AND logic) for a profile to sync.

5. **Logging Strategy**: Use CloudWatch with the integration key (account/section/integration) to locate logs. Delta Sync Manager produces messages; Delta Sync Worker processes them and logs validation errors.

6. **Queries and Profiles**: Imports external system queries as Apsis tags. Useful for segmentation. Manual import available; recurring option for scheduled syncs.

7. **Generic Connector Approach**: New external systems must conform to a standard specification. Postman collection available for validation before implementation.

8. **Field Validation**: Phone numbers and other fields are validated during worker processing. Invalid data is skipped; validation failures appear in Delta Sync Worker logs.

---

## Unresolved Questions and Follow-Up Items

1. **API Key Creation in FEC 12.1**: How to create new integration API keys in the external CRM system. [Michal Rosikiewicz noted the team lacks permissions; may require enterprise team assistance.]

2. **Custom Query Creation**: Building custom queries in the external FEC system. [Shreevidhya Ganesan deferred to Eric for detailed explanation in a future session.]

3. **Outbound Consent Sync Timing**: Confirmation of why consent changes from Apsis → CRM took longer than expected (likely batch processing, but not explicitly confirmed at end of session).

4. **Workshop on New External System Onboarding**: [Lukasz Grabowski proposed a dedicated workshop on the full process of integrating a new external CRM from requirements to production. Shreevidhya Ganesan agreed it would be valuable.]

5. **Deep Linking Architecture**: Why deep linking prevents multiple installations for Dynamics/Shopify. [Mentioned briefly; Eric may elaborate in architecture discussion.]