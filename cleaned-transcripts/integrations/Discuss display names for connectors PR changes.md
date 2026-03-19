---
source_file: Discuss display names for connectors PR changes.txt
domain: Integrations
topics: [Display Names vs Integration IDs, Connector Configuration, Folder Creation in Apsis, Lead Entity Handling, Additional Entities Configuration, Data Migration Strategy]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Apsis One, Connector Framework, lib/connectors, installer.go, Additional Entities, Lead Entities, FSC Enterprise, FSC Corporate, E.Deal, Dynamics, Tribe]
session_type: knowledge-transfer
---

## Session Overview

This session discusses a PR that changes how folder names are created in the Apsis One integration platform. The core issue is that the system was previously using internal **integration IDs** (logical names) for folder creation, but should instead use **display names** (customer-visible names). The discussion covers why this distinction matters, how connector configuration works, which integrations support additional entities like leads, and the strategy for migrating existing folders with incorrect names.

---

## Understanding Display Names vs Integration IDs

### The Core Problem

[Erik Andersson]: When we create folders in Apsis One in the integration platform, we should utilize the display name instead of the logical name. This is a distinction between what we call something internally versus what customers see.

**Integration ID** (logical name): The internal identifier used by the system. Examples include `FSC_Enterprise_2`, `FSC_Corporate`. These are:
- Part of the agreed-upon protocol and API contract between us and the CRM system
- Extremely difficult to change because any reference to them in the database would require massive data migration
- Not visible to customers
- Used in system logs and internal documentation

**Display Name**: The customer-visible label shown in the UI and in Apsis One folder names. Examples include `FSC Enterprise 12.1`, `E.Deal`, `12.0`. These are:
- Easy to change in the system
- Visible to customers in the folder structure
- Subject to rebranding and localization needs

### Real-World Example: E.Deal vs FSC Corporate

[Erik Andersson]: E.Deal was previously called FSC Corporate internally, and it was only recently renamed to E.Deal. The integration ID remains `FSC_Corporate` to avoid data migration, but the display name was changed to `E.Deal`.

This means:
- Customers see `E.Deal` in the UI
- Internal logs and configurations still reference `FSC_Corporate`
- Folders created before the display name change still have the old name `FSC_Corporate`
- Future folders will be created with the display name `E.Deal`

---

## Connector Configuration: Where Display Names Live

### File Structure

Connector configuration is found in the `lib/connectors` directory. For each connector, there is typically an `installer.go` file (or `connector.go` for very old connectors like Dynamics). This file contains initialization and store options.

[Erik Andersson]: Inside the installer.go file, you'll find a section with `init` and `store option` where the **integration ID** and **display name** are both defined.

Example structure for FSC Enterprise 2:
```
Integration ID: FSC_Enterprise_2
Display Name: FSC Enterprise 12.1
```

The display name is what appears to customers, while the integration ID is the internal reference.

---

## Folder Creation in Apsis One: Two Main Cases

### Case 1: Lead Entity Handling

When an integration that supports lead gathering is installed, the system creates a dedicated **lead folder** in Apsis One to handle the lead entity separately from the main contact entity.

[Erik Andersson]: By default, integrations support contacts or persons—the main customer entity in the CRM system. But we also support lead gathering. For that, we create a dedicated lead folder by creating a dedicated folder.

**Why this matters**: Some CRM systems have a distinct concept of leads (prospects) separate from contacts (existing customers). We need to map these separately in Apsis.

**The problem this PR fixes**: Previously, this lead folder was created using the integration ID instead of the display name. So if E.Deal was installed with lead support, a folder called `FSC_Corporate` would be created instead of `E.Deal`.

### Case 2: Subscription Sync (Maxo Example)

[Erik Andersson]: Maxo implemented a system with real-time sync for subscriptions—not consent itself, but what they had consent in. If they created a consent list called "technical newsletters," they sent us a webhook and we would create a subscription folder under the integration ID name.

Example: If E.Deal supported this flow, a folder would be created called `FSC_Corporate` under subscriptions instead of `E.Deal`.

**Note**: Maxo was the only system that implemented this subscription sync flow.

---

## Additional Entities Configuration

### How to Identify Which Integrations Support Which Entities

In the `installer.go` files, look for the **additional_entities** section. This tells you which integrations support entities beyond the main contact/person entity.

[Erik Andersson]: You can see code-wise which integrations have additional entities configured. You'll often see a lead entity here with:
- The ID for that entity
- The name of it
- What field on the profile it uses as the unique identifier
- What you can do with it

### Integrations with Lead Support

The following integrations support lead entities:
- **FSC Corporate (E.Deal)**: Has a "Silhouette" entity representing leads
- **Dynamics** (new version): Supports lead entity
- **Tribe**: Supports lead entity
- **FSC Enterprise 2**: Does NOT support leads (they only create contacts)
- **FSC Enterprise** (12.0): Does NOT support leads

[Erik Andersson]: If you were to install Dynamics by Sideshop, FSC Enterprise 2, or Tribe, you will see this additional lead folder. But Maxo supported the flow with real-time subscription sync, so this folder will not appear anymore for any existing Maxo systems.

### Other Potential Additional Entities

While lead entities are the most common additional entity, the system theoretically supports other entity types such as:
- Company entities
- Organizational entities

However, there isn't a good way of representing these in Apsis currently, so they aren't used in practice.

---

## The PR Changes: What Was Modified

### What Changed

The PR modifies the code to use **display names** instead of **integration IDs** when creating folders in Apsis One.

Code change summary:
- Renamed `FSC_Corporate` to `E.Deal` (display name)
- Renamed `FSC_Enterprise_2` to `FSC Enterprise 12.1` (display name)
- Renamed `FSC_Enterprise` to `FSC Enterprise 12.0` (display name)

**Critical**: The integration IDs themselves were NOT changed. Only the display names were updated.

### What This Means Going Forward

[Erik Andersson]: In the future, when you install a folder, we will create it with the actual display name, so if someone decides they want to change it from E.Deal to something else, you only need to change the display name in the integration configuration and then this will be propagated to any other service which references it.

**Important caveat**: Already-created folders with the old incorrect name will keep the old name. Fixing them would require a data migration, but we will not create new folders with the old incorrect name.

---

## Data Migration Strategy for Existing Folders

### Scope of the Problem

Currently, there are only 2-3 E.Deal customers, so migration is feasible.

### Proposed Approach

[Michal Rosikiewicz]: If it's a simple call to Apsis, we can have a script to run the migration rather than manual API calls.

The migration would:
1. Extract the folder IDs from the affected accounts
2. Make API calls to Apsis One to rename the folders from `FSC_Corporate` to `E.Deal`
3. Run against production accounts

### Access and Permissions Considerations

[Erik Andersson]: We need access to customer accounts first and foremost. For CRM customers we work with, someone collaborates with us and we have standing consent to do whatever we need. We just need to ask SOC to give us access to it.

[Michal Rosikiewicz]: This approach of asking clients to give access to their account just for one job to update folder names seems inefficient.

**Better approach** (internal delegated keys): Instead of requesting customer account access, use internal delegated keys from the identity provider. This allows migrations to be run without requiring customer permission.

[Michal Rosikiewicz]: We can create internal delegated keys from IO and be able to do the migration without asking the customer for access.

[Erik Andersson]: I don't put too much weight on which way we do it. If we have an approved approach using internal keys, I'm very happy to do that instead.

---

## System Visibility of Integration IDs vs Display Names

### Where Integration IDs Are Exposed

Integration IDs appear in:
- System logs
- API contracts with CRM systems
- Internal database references
- Integration keys (the "holy trinity" of account section, logical name, and connector)

These are NOT visible to customers.

### Where Display Names Are Exposed

Display names appear in:
- UI on the integration page (what customers see as available connectors)
- Folder names in Apsis One (what customers see in their account)
- Any UI elements showing connector branding

This is what customers interact with directly.

[Erik Andersson]: This folder name is exposed to the customer, so we need to utilize the customer-known name for it. This is also relevant for any potential translation cases.

---

## PR Approval and Code Location

### Change Size and Approval Status

[Erik Andersson]: This is not a big change. We have just changed from using the integration ID to utilizing the display name, and then we have also modified the options for it. I have already gone ahead and approved this PR because it was quite small.

### Key Implementation Detail

The critical thing to remember is:
- **Where**: The configuration is in `lib/connectors` in the `installer.go` files
- **What impact**: Understanding what configuration options are used for in the code and how they affect Apsis

---

## Future Work: Leads Knowledge Transfer

[Erik Andersson]: We'll cover leads in more detail at a future meeting. Topics to be discussed include:
- How leads are gathered
- How this works with outbound mappings
- How we handle the merging when a lead becomes a fully-fledged contact
- How we keep track of lead profiles vs contact profiles in Apsis

[Michal Rosikiewicz]: The key to finding this information in code is looking for **additional_entities** in the connector configuration.

---

## Key Takeaways

1. **Display names** (customer-visible) and **integration IDs** (internal logical names) serve different purposes. Display names should be used for anything the customer sees, especially folder names in Apsis One.

2. **Integration IDs are hard to change** because they're part of the API contract and would require data migration. Display names are easy to change and should be updated when branding or naming changes occur.

3. **Connector configuration** lives in `lib/connectors/installer.go` files. This is where both display names and integration IDs are defined, and it controls what gets created in Apsis One.

4. **Additional entities** (primarily leads) are configured per-connector. Not all integrations support leads. Check the `additional_entities` section in the installer.go file to see what's supported.

5. **The PR changes folder creation logic** to use display names going forward, ensuring customer-visible folder names match current branding. Existing folders with old names would require a data migration (currently planned only for 2-3 E.Deal accounts).

6. **For migrations**, use internal delegated keys from the identity provider rather than requesting customer account access. This is the approved approach going forward.

7. **Lead handling** is complex and will be covered in a separate knowledge transfer session. Understand the distinction between main entities (contacts) and additional entities (leads) when reviewing connector configs.

---

## Unresolved Questions & Action Items

- **Action**: Create a story for the E.Deal folder migration using internal delegated keys approach (assigned to team backlog planning discussion)
- **Future Discussion**: Detailed knowledge transfer on lead entity handling, including gathering, outbound mappings, and lead-to-contact conversion logic
- **Clarification Needed**: Timeline and prioritization of the E.Deal migration relative to other backlog items (to be discussed in sprint planning/daily standup)