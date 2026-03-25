---
source_file: Discuss display names for connectors PR changes.txt
domain: Apsis One Integrations
topics: [Display Names vs Integration IDs, Folder Creation in Apsis One, Connector Configuration, Lead Entity Handling, Data Migration Strategy]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Apsis One, Integration Platform, Connector Configuration (installer.go, connector.go), Display Names, Integration IDs, Lead Attributes, Subscription Folders, Additional Entities]
session_type: architecture-review
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, Tribe Integration, E-deal Integration, Lead creation]
---

## Session Overview

This session reviewed a pull request that changes the connector folder creation logic in Apsis One to use **display names** instead of **integration IDs** when creating folders. The key insight is that while integration IDs are internal logical names that are difficult to change (they're part of the API contract), display names are customer-facing and can be easily updated. The PR affects folder creation in two scenarios: when installing integrations that support leads (creating a dedicated lead attribute folder) and when setting up real-time subscription sync (as implemented for Maxo). The team discussed the implications of this change, which connectors support leads, and planned a potential data migration for existing E-deal customers.

---

## Understanding Display Names vs Integration IDs

### The Problem: Hidden Complexity in Naming

**Display names** and **integration IDs** can diverge, creating confusion between what customers see and what the system internally uses. 

[Erik Andersson]: E-deal provides a concrete example. The connector was originally called "FSC Corporate" internally but has since been renamed to "E-deal" for customer-facing purposes. However, the internal integration ID remains unchanged because:

1. **The integration ID is part of the agreed-upon protocol and API contract** between Apsis One and the CRM system
2. **Changing the integration ID requires massive data migrations** everywhere the ID is referenced in the system

In contrast, **changing the display name is trivial** — it only affects the UI label and has no backend implications.

### Real-World Example: Efficy Enterprise

On the Apsis One integration page, customers see "Efficy Enterprise 12.0" and "Efficy Enterprise 12.1". However, internally (in logs and code), these connectors are referenced as:
- `Efficy Enterprise` (logical integration ID)
- `Efficy Enterprise 2` (logical integration ID)

This mismatch existed because folder creation previously used the integration ID instead of the display name, making internal folder names visible to customers.

### Impact on Folder Visibility

[Erik Andersson]: When you look at a customer's profile and see folders associated with an integration, **those folder names are customer-visible and must use the display name, not the integration ID**.

> "This folder name is because the integration ID was used on creation and it would have been the same here. So the whole point of this PR is that the display name should be utilized when we create the folder and not the integration ID because this is actually something which the customer can see."

---

## When Folders Are Created in Apsis One

### Case 1: Lead Support in Integrations

When an integration is installed that supports **lead gathering**, Apsis One creates a dedicated **lead ID attribute** folder.

[Erik Andersson]: The reason is straightforward: by default, integrations support the main customer entity (contacts or persons in CRM systems). However, many CRM systems have a separate lead entity. To support lead gathering:

1. A dedicated lead attribute is created
2. This is done by creating a dedicated folder
3. Previously, this folder was named using the integration ID; now it uses the display name

**Which connectors support leads?**
- **Dynamics** (multiple versions support the lead entity)
- **Tribe** (supports a lead entity)
- **Efficy Enterprise 12.1** (no concept of leads; creates contacts only, not a separate lead entity)
- **Efficy Enterprise 12.0** (no leads support)
- **E-deal/FSC Corporate** (supports a "Silhouette" entity representing leads; configured as an additional entity)

**Which connectors do not support leads?**
- **Efficy Enterprise 12.0** — sends form submissions as contacts directly
- **Efficy Enterprise 12.1** — sends form submissions as contacts directly

### Case 2: Real-Time Subscription Sync (Maxo Example)

[Erik Andersson]: Maxo implemented a system for real-time sync of consent subscriptions (not just consent status, but what subscription list the consent applies to).

When Maxo creates a consent list (e.g., "technical newsletters"), it sends a webhook to Apsis One. Previously, Apsis One would create a subscription folder under the **integration ID name**. For E-deal, this would have created a folder called "FSC Corporate" under subscriptions. With the PR change, it now uses the display name.

> "Only Maxo supported the flow with the real time subscription sync here. So this folder will not appear anymore for any existing Theorem systems. But this folder will appear for quite a lot of them."

[Erik Andersson clarifies]: The subscription sync folder will not appear for other systems going forward because they don't have this capability, but any system with additional entities (like leads) will see a folder created.

---

## Connector Configuration: Where Display Names Live

### File Locations

Connector configuration is stored in the `lib/connectors` directory. The exact filename varies by connector age:

- **Newer connectors**: `installer.go` or `connector.go`
- **Oldest connectors** (e.g., Dynamics): May have configuration in a different location but follow the same pattern

### Configuration Structure

Each connector has an `InitStoreOption` section containing:

- **Integration ID**: The internal logical name (e.g., `FSC_ENTERPRISE_2`, `EFFICY_ENTERPRISE_2`)
- **Display Name**: The customer-facing name (e.g., "E-deal", "Efficy Enterprise 12.1")
- **Additional Entities**: Configuration for secondary entities like leads

### Example Configuration

For E-deal (formerly FSC Corporate):
- Integration ID: `FSC_CORPORATE` (unchanged, part of API contract)
- Display Name: Was "FSC Corporate", now "E-deal" (can be changed freely)

For Efficy Enterprise versions:
- Integration ID: `EFFICY_ENTERPRISE_2` and `EFFICY_ENTERPRISE` respectively
- Display Name: "Efficy Enterprise 12.1" and "Efficy Enterprise 12.0"

For E-deal's lead support:
- Additional entity configured as "Silhouette" (their representation of leads)
- Entity ID, name, and unique identifier field are all configured

### Finding Lead Configuration in Code

To determine if a connector supports leads, look for **additional entities** configuration in the `installer.go` file:

```
additional_entities {
  entity_id = "..."
  entity_name = "..."
  unique_identifier_field = "..."
}
```

If an additional entity entry exists, that entity (often leads) will have a dedicated folder created in Apsis One.

---

## Impact of the PR: What Changes, What Doesn't

### What Changed

The PR modifies folder creation logic to use the display name instead of the integration ID when:
1. Creating lead attribute folders for integrations with lead support
2. Creating subscription folders for integrations with real-time sync capabilities

### Specific Changes in the PR

- Changed folder creation from `FSC_CORPORATE` to `E-deal`
- Changed folder creation from `EFFICY_ENTERPRISE_2` to `Efficy Enterprise 12.1`
- Changed folder creation from `EFFICY_ENTERPRISE` to `Efficy Enterprise 12.0`

### What Does NOT Change

- **Integration IDs remain identical** — they are not touched by this PR
- **Internal logs and API references** still show integration IDs (e.g., `EFFICY_ENTERPRISE_2`)
- **The API contract** with CRM systems is unaffected

[Erik Andersson]: 

> "So here you can see we changed it from FSC Corporate to E-deal FC Enterprise 2 to 12.1 and FC Enterprise to 12.0. But again this does not mean that we have touched the integration ID."

Integration keys in logs still use the logical name. For example, in integration logs you'll see:
```
integration_id = "EFFICY_ENTERPRISE_2"
integration_key = [account_section + logical_name]
```

The folder name exposed to customers is separate and now uses the display name.

### Future-Proofing Benefits

[Erik Andersson]: Once deployed, this change makes future display name changes trivial:

1. If a display name needs to change again (e.g., E-deal is renamed), only the `DisplayName` field in the connector configuration needs updating
2. This change automatically propagates to any service that references the display name
3. **Existing folders retain their old names** (this is just how it works; they don't automatically get renamed)
4. **Only newly created folders** will use the new display name
5. If a complete rename of all existing folders is needed, a separate data migration is required

---

## Data Migration Considerations for Existing Customers

### E-deal Migration Status

Currently there are approximately **2-3 E-deal customers**. The decision on whether to migrate existing folder names hasn't been made yet and requires discussion with Emmanuel (E-deal product owner).

[Michal Rosikiewicz]: Given the small number of customers, a manual migration approach using API calls is viable.

[Erik Andersson]: A migration script could extract folder IDs and rename them via the Apsis One API. This would be preferable to asking customers for direct account access.

### Previous Approach (No Longer Acceptable)

The old approach involved:
1. Asking customers to grant direct account access
2. Using customer tokens to make updates
3. This is **no longer acceptable** due to compliance and security changes

[Michal Rosikiewicz]: 

> "For me, it's something we should avoid. Even if it's still only for three accounts."

### New Proposed Approach: Internal Delegated Keys

[Michal Rosikiewicz]: Apsis One now supports **internal delegated keys** issued by the IO (Apsis One) team, allowing:

1. Internal migrations without requesting customer account access
2. A script can extract folder IDs and run a migration
3. No need to ask customers for permission or credentials

[Erik Andersson]: This is preferable. If an approved internal delegation approach exists, it should be used.

### Next Steps

[Michal Rosikiewicz]: Plans to create a story (backlog item) for the E-deal folder name migration using internal delegated keys. This would be coordinated with Erik, Prem, and potentially Lukasz, depending on sprint prioritization.

The migration has not been scheduled yet — it's pending:
1. Product owner (Emmanuel) sign-off
2. Availability of the team for implementation
3. Prioritization relative to customer-impacting issues

---

## Deeper Context: How Lead Entities Work Across Different CRM Systems

### The Generic Connector Model

[Erik Andersson]: The integration platform uses a **generic connector approach** where:

1. Every CRM system integration defines one **main entity** (typically contacts or persons)
2. Additional entities (like leads, company entities, organizational entities) can be configured
3. Form submissions typically target the main entity, but can also create additional entity records

### System-Specific Implementations

**Efficy Enterprise 12.0 & 12.1**:
- Main entity: Contacts
- Lead support: None — they create contacts for everything
- When Apsis One submits forms, they create pure contact records, not separate leads

**E-deal/FSC Corporate**:
- Main entity: Contacts
- Lead support: Yes, via the "Silhouette" entity
- Silhouette is configured as an additional entity with specific field mappings

**Dynamics**:
- Lead entity support: Available (varies by Dynamics version)
- Additional entity configuration includes lead-specific settings

**Tribe**:
- Lead entity support: Yes
- Configured as an additional entity

**Maxo**:
- Real-time subscription sync for consent lists
- Different from lead handling; specific to subscription management

### Why Lead Support Matters

Leads represent potential customers who haven't yet been fully qualified as contacts. Some CRM systems maintain strict separation between leads and contacts, requiring Apsis One to:

1. Create dedicated lead folders in Apsis One
2. Track which CRM lead corresponds to which incoming form submission
3. Handle the eventual conversion of a lead to a fully-fledged contact (covered in a future knowledge transfer session)

[Erik Andersson]: The broader lead handling process — including gathering, merging, and eventual lead-to-contact conversion — will be discussed in detail in a future knowledge transfer session.

---

## Key Takeaways

1. **Display names and integration IDs serve different purposes**: Display names are customer-facing and easily changeable; integration IDs are internal, part of the API contract, and very expensive to change.

2. **Folder names in Apsis One are customer-visible** and must use display names, not integration IDs. This PR fixes that.

3. **Two scenarios trigger folder creation**:
   - When an integration supports leads (creates a dedicated lead attribute folder)
   - When an integration supports real-time subscription sync (Maxo example)

4. **To find what lead support a connector has**, look in `lib/connectors/[connector]/installer.go` (or similar) for the **additional_entities** configuration section.

5. **The PR is backward compatible**: Existing folders keep their old names; only new installations create folders with the new display name.

6. **Future display name changes are now trivial** — just update the configuration, and new folders will reflect the change. Old folders remain unchanged unless explicitly migrated.

7. **Data migration for existing customers** (like the 2-3 E-deal customers) requires:
   - Product owner approval
   - Internal delegated keys (the new, preferred approach) instead of requesting customer account access
   - A script to rename folders via the Apsis One API

8. **Lead handling is complex**: It involves gathering, tracking, merging, and eventual conversion to contacts. This is a separate knowledge transfer topic.

---

## Unresolved Questions & Action Items

### Pending Decisions
- **E-deal folder name migration**: Needs discussion with Emmanuel (product owner) to decide if 2-3 existing customers should have their folder names migrated from "FSC Corporate" to "E-deal"

### Planned Action Items
- **[Michal Rosikiewicz]**: Create a backlog story for E-deal folder name migration using internal delegated keys (to be assigned to Erik, Prem, and/or Lukasz pending sprint prioritization)
- **Future knowledge transfer session**: Deep dive into lead entity handling, including how leads are gathered, how they work with outbound mappings, and how lead-to-contact conversion is tracked in Apsis One

---