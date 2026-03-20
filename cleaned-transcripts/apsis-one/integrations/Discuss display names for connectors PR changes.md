---
source_file: Discuss display names for connectors PR changes.txt
domain: Apsis One - Integrations
topics: [Display Names vs Integration IDs, Connector Configuration, Folder Creation in Apsis, Lead Entity Handling, Data Migration Strategy]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Integration Platform, Connectors Library, Installer Configuration Files, Lead Attributes, Subscription Folders, CRM Integrations (FSC Enterprise, E-Deal, Dynamics, Tribe), Apsis One Folder Structure]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covers a PR that changes how folder names are created in the Apsis One integration platform. The core issue is that previously, folders were created using the **integration ID** (the internal logical name), but they should be created using the **display name** (the customer-facing name). The discussion explains why this distinction matters, where the configuration lives in the codebase, the impact on customer-visible artifacts, and future migration plans for existing data.

---

## The Problem: Integration IDs vs Display Names

### The Core Distinction

[Erik Andersson]: When we create folders in Apsis One in the integration platform, we should utilize the display name instead of the logical name.

The system maintains two separate identifiers for each connector:

- **Integration ID (Logical Name)**: The internal identifier used in code and API contracts. Examples: `FSC Enterprise 2`, `FSC Corporate`
- **Display Name (Customer-Facing Name)**: What customers see in the UI. Examples: `FSC Enterprise 12.1`, `E-Deal`

These names can diverge. For example, the connector historically called `FSC Corporate` was renamed to `E-Deal` for customers, but the internal integration ID remained `FSC Corporate`.

### Why This Distinction Matters

[Erik Andersson]: The display name is very, very easy for us to change in the system, but the logical integration ID is, on the contrary, very hard for us to modify in the system because:
1. The integration ID is part of the agreed upon protocol and API contract between us and the CRM system
2. Changing it would require a massive data migration everywhere we reference or utilize the integration ID

Changing the display name in the UI is trivial compared to modifying the integration ID, which would cascade through the entire system.

---

## Where Folders are Created in Apsis One

### Two Primary Use Cases

[Erik Andersson]: There are two cases where we create folders in Apsis One:

#### Case 1: Lead Entity Support
When we install an integration that supports handling leads, we create a dedicated **lead ID attribute** inside Apsis One. This is necessary because:
- Integrations support contacts/persons (the main customer entity) by default
- Lead gathering requires a dedicated lead entity representation
- We create a dedicated folder for this purpose

Previously, this folder was named using the integration ID. For example, if `FSC Corporate` supported leads, we would create a folder called `FSC Corporate` under the leads section, but it should have been named `E-Deal` (the display name).

#### Case 2: Real-Time Subscription Sync (Maxo)
[Erik Andersson]: For Maxo, we implemented a system where we had real-time sync for subscriptions. Not consent itself, but what consent lists they actually have. So if they created a consent list called "technical newsletters," they sent us a webhook and we would create a subscription folder under the integration ID name.

If E-Deal had supported this flow, we would have created a folder called `FSC Corporate` under subscriptions with the consents, but it should have been `E-Deal`.

---

## Connector Configuration Structure

### Location in Codebase

[Erik Andersson]: In the `lib/connectors` folder, each connector has a configuration file. For newer connectors it might be `installer.go` or `connector.go`. For the absolutely oldest ones like Dynamics, there will be one section where it shows the `init` and `store option`.

Example file path structure:
```
lib/connectors/
├── FSC_Enterprise_2/
│   └── installer.go
├── FSC_Corporate/
│   └── installer.go
└── Dynamics/
    └── installer.go
```

### Configuration Example

In the `installer.go` file, you will find:

- **Integration ID**: `FSC Enterprise 2` (the internal logical name)
- **Display Name**: `FSC Enterprise 12.1` (what customers see)

The integration ID is the identifier used in the agreed protocol with the CRM system, while the display name is what gets exposed in the customer-facing UI and, with this PR, in the folder names.

---

## Impact: How Display Names Affect Customer-Visible Artifacts

### The Integration Page Display

On the integration page, customers see only display names and versions. For example, customers see:
- `FC Enterprise 12.0`
- `FC Enterprise 12.1`

Behind the scenes, the internal names are:
- `Efficy Enterprise`
- `Efficy Enterprise 2`

### Folder Names in Apsis One (Customer-Exposed)

The critical issue is that **folder names in Apsis One are exposed to customers**. These are not internal implementation details.

[Erik Andersson]: When we changed the display name to E-Deal and FC Enterprise 12.1, this is something the customer can see. The folder name is exposed to the customer, so we need to utilize the actual display name, not the integration ID.

Before this PR: If a customer installed an integration, the folder would be named using the integration ID (e.g., `FSC Corporate`), which didn't match what they saw on the integration page (e.g., `E-Deal`).

After this PR: Folders will be created with the display name, ensuring consistency between what customers see in the UI and what appears in their folder structure.

---

## The PR Changes: Migration to Display Names

### What Changed

[Erik Andersson]: Here you can see we changed it from:
- `FSC Corporate` → `E-Deal`
- `FSC Enterprise 2` → `12.1`
- `FSC Enterprise` → `12.0`

The changes involve:
1. Modifying the code to use the display name when creating folders instead of the integration ID
2. Updating configuration options that reference these names

**Important caveat**: This does not mean the integration ID has been touched. The integration ID remains unchanged in the API contracts and logs.

### Code-Level Changes

In the relevant code sections, we changed from passing the `integration_id` parameter to passing the `display_name` parameter when folder creation is triggered.

---

## Future Implications: Forward-Looking Behavior

### New Installations
[Erik Andersson]: In the future when you install a folder, we will create it with the actual display name. So if someone decides that they don't want to keep the name as E-Deal and wants to re-change it to something else, then you only need to change the display name in the integration configuration, and this will be propagated to any other service which references it.

### Existing Folders
[Erik Andersson]: Already created folders will still have the old name. That would require its own data migration, but we would not create new folders with the old incorrect name.

This approach minimizes risk: only future installations benefit from the fix, while existing customer data is left untouched unless explicitly migrated.

---

## Understanding Additional Entities and Lead Configuration

### Configuration in Code

[Erik Andersson]: You always configure one main entity (contacts, persons, or whatever the CRM has), and then anything on top of that might be leads. You can see which integrations support leads by looking at the code.

In the connector configuration, you'll see something like:
```
additional_entities:
  - entity_id: "lead_entity_id"
    entity_name: "Lead"
    unique_identifier_field: "field_name"
```

### Which Integrations Support Leads

[Erik Andersson]: Integrations that support lead entities include:
- **FSC Corporate**: Has a "Silhouette entity" which is their representation of leads (configured under `additional_entities`)
- **Dynamics by SideShop**: Supports lead entities
- **Tribe**: Supports a lead entity

Integrations that do **not** support leads:
- **FSC Enterprise 12.0**: Has no lead configured because they don't have any concept of leads. They create contacts in their CRM system. When we submit forms to them, they send contacts to us and gather submissions as pure contacts.

### Real-Time Subscription Sync (Maxo-specific)

[Erik Andersson]: Only Maxo supported the flow with the real-time subscription sync. This folder will not appear anymore for any existing third-party systems, but this folder (for leads) will appear for quite a lot of them.

The subscription sync feature created folders dynamically based on webhook events from Maxo indicating new consent lists. This is distinct from the lead entity folder, which is static based on connector configuration.

---

## Data Migration Strategy for Existing Customers

### Current Status and Constraints

[Michal Rosikiewicz]: We haven't planned a migration yet. I haven't discussed it with Emmanuel, who is the product owner for E-Deal. It might very well be that we will do a migration because we don't have that many E-Deal customers. I think we have two or three right now.

### Proposed Migration Approach

Two approaches were discussed:

#### Approach 1: Manual API Calls via Customer Account Access (Previously Used)
[Erik Andersson]: We can just do that with manual API calls to Apsis. We would ask SOC to give us access to the customer account, then make API calls to rename the folders.

**Concerns**: This requires asking customers for account access just to perform a folder rename, which is not ideal.

#### Approach 2: Internal Delegated Keys (Preferred)
[Michal Rosikiewicz]: For me, it's something we should avoid asking clients to give us access to the account just to have one job that we'll use to update the name of the folder. We can use internal delegated keys from Apsis One and we'll be able to do this migration with those keys.

[Erik Andersson]: If we have an approved approach where we use internal keys, I am very happy to do that instead.

**Key Insight**: The system previously used system keys for access, but this is no longer supported. The new preferred approach is to use internal delegated keys, which is less intrusive than asking customers for account access.

### Planning Next Steps

[Michal Rosikiewicz]: I'll create a story for the migration. This is something we can plan and prioritize.

The migration is considered a separate task from the PR itself, to be planned and executed after the display name changes are merged.

---

## Integration ID vs Display Name in Logs and Internal Systems

### Where Integration IDs Appear

[Erik Andersson]: In the logs, you have the integration ID. For example, if it's for FSC Enterprise 12.1, it would say `FSC Enterprise 2` because that is our logical name. The integration key looks like: `[account_section].[logical_name_for_the_connector]`. But again, this is not exposed to the customer.

### The Critical Difference

[Erik Andersson]: The integration key is not exposed to the customer, whereas the folder is exposed to the customer. So we need to utilize the customer-known name for it. This is also relevant for any potential translation cases.

Internal logs and API contracts will continue to use the integration ID for stability and to avoid breaking changes in the protocol. Only customer-facing artifacts (folders, UI display names) use the display name.

---

## Key Takeaways

1. **Display names are customer-facing identifiers** that should be used for any artifact visible to customers (folder names, UI labels), while integration IDs are internal logical names used in API contracts and logs.

2. **Display names are easy to change, integration IDs are not**. Changing the display name is a UI/config update. Changing the integration ID requires massive data migrations and breaks API contracts.

3. **This PR ensures consistency**: Customers now see the same name in the integration page and in their folder structures.

4. **Configuration lives in `lib/connectors/[connector_name]/installer.go`** (or similar). Look for the `init` and `store option` sections to find integration ID and display name.

5. **Additional entities (like leads) are configured per-connector** and determine what folders get created. Check the `additional_entities` section in the configuration to understand which connectors support leads and subscription syncing.

6. **Forward-compatible approach**: Existing folders keep their old names (requires separate migration), but all new installations will use the correct display name.

7. **Lead folder creation happens in two scenarios**: 
   - When an integration supports leads as an additional entity (most connectors)
   - When an integration supports real-time subscription sync via webhooks (Maxo only)

8. **Migration strategy should use internal delegated keys** rather than asking customers for account access, avoiding unnecessary access requests for routine maintenance tasks.

---

## Unresolved Questions and Action Items

- **Action Item**: [Michal Rosikiewicz] Create a story for the data migration of existing E-Deal customer folders from `FSC Corporate` to `E-Deal` using internal delegated keys
  
- **Future Knowledge Transfer Session**: [Erik Andersson] and [Michal Rosikiewicz] plan to cover in more detail how leads are gathered, how this works together with outbound mappings, and how lead-to-contact merging is handled in Apsis One (keeping track of both lead and contact profiles)

- **Backlog Prioritization**: Discussion deferred to daily standup or after-standup. Erik mentioned there are customer issues and potential pain points that should be addressed before team members depart. Michal expressed openness to reviewing or grabbing stories from the backlog.