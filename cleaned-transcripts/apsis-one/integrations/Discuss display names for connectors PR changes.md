---
source_file: Discuss display names for connectors PR changes.txt
domain: Apsis One Integrations
topics: [Display Names vs Integration IDs, Folder Creation in Apsis One, Lead Entity Configuration, PR Changes and Code Structure]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Connector Configuration Files, installer.go, Display Names, Integration IDs, Lead Attributes, Subscription Folders, Additional Entities]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Efficy Enterprise 12.0, Efficy Enterprise 12.1, E-deal (Efficy Corporate), Lead creation]
---

## Session Overview

This knowledge transfer session covers a pull request that changes how Apsis One creates folders during connector installation. The key change is shifting from using **integration IDs** (internal logical names) to **display names** (customer-visible names) when creating folders. The discussion explores why this distinction matters, where folder creation happens in the codebase, how lead entities are configured per connector, and the implications for data migration in cases where display names have changed (such as the FSC Corporate → E-deal rename).

---

## Display Names vs. Integration IDs: The Core Distinction

### Why This Matters

[Erik Andersson]: The fundamental problem is that **display names** and **integration IDs** (logical names) can diverge over time. Display names are what customers see in the UI and in Apsis One; integration IDs are the internal identifiers used in the system.

For example:
- **E-deal**: Historically called "FSC Corporate" internally, but rebranded to "E-deal" in the customer-facing display name
- **Efficy Enterprise**: Displayed as "12.0" and "12.1" to customers, but internally identified as "Efficy Enterprise" and "Efficy Enterprise 2"

### Why Integration IDs Are Hard to Change

> Integration IDs are part of the agreed-upon protocol and API contract between us and the CRM system. Changing them would require massive data migration everywhere we reference them.

In contrast, display names are trivial to change in the system—it's just a UI label update.

### The Consequence of Using Integration IDs for Folder Creation

When folders were created using integration IDs instead of display names, customer-visible folders in Apsis One contained outdated or internal names. For instance, if E-deal was still internally identified as "FSC Corporate" at folder creation time, the folder would be labeled "FSC Corporate" even though the integration was rebranded to "E-deal" in all customer-facing UI.

---

## Where Folder Creation Happens: Two Primary Cases

### Case 1: Lead Entity Creation During Integration Installation

[Erik Andersson]: When an integration that supports lead handling is installed, Apsis One creates a dedicated **lead ID attribute** and accompanying folder structure.

Most integrations support the main customer entity (contacts, persons, or similar primary CRM entities), but some also support leads as a secondary entity type. A dedicated lead folder is created for these integrations.

**Example connectors with lead support:**
- **Dynamics** (Microsoft Dynamics)
- **Tribe**
- **FSC Corporate** (has a "Silhouette" entity representing leads)
- **Efficy Enterprise 12.0 and 12.1** do NOT support leads (they create contacts directly)

### Case 2: Real-Time Subscription Sync (Maxo)

[Erik Andersson]: Maxo implemented a system for real-time subscription synchronization. When a customer creates a consent list (e.g., "technical newsletters"), Maxo sends a webhook. In response, Apsis One creates a **subscription folder** under that consent list name.

If E-deal had supported this flow, a folder would have been created named "FSC Corporate" (or now "E-deal") under subscriptions, but the subscription sync feature is currently only implemented for Maxo.

---

## Connector Configuration: Where Display Names Are Defined

### File Structure: `lib/connectors/`

Each connector has a configuration file (typically `installer.go` or `connector.go` for older connectors like Dynamics) containing an **init store option** section.

**Example from FSC Enterprise 2:**

```
Integration ID: FSC Enterprise 2 (internal logical name)
Display Name: FSC Enterprise 2 (initially; now changed to "E-deal" for FSC Corporate, "12.1" for Efficy Enterprise 2, etc.)
```

[Erik Andersson]: You can find the configuration by looking in the connector's `installer.go` file. This is where both the integration ID and display name are declared. The integration ID never changes; the display name can be updated as needed.

### Additional Entities Configuration

Beyond the primary entity (contacts/persons), connectors can have **additional entities** configured. These are visible in the code as:

```
Additional entities:
  - Entity ID
  - Entity name
  - Field used as unique identifier
  - Additional configuration
```

Lead entities appear here when supported. To determine which integrations support leads, search the codebase for "additional entities" configuration in each connector's installer file.

---

## The PR Change: From Integration IDs to Display Names

### What Changed

The PR modifies the folder creation logic to use `display_name` instead of `integration_id` when creating folders in Apsis One.

[Erik Andersson]: The change is small but important. Instead of:
```
folder_name = integration_id
```

We now use:
```
folder_name = display_name
```

### Impact on Existing vs. Future Installations

**Future installations:** Folders will be created with the current display name. If a display name is changed in the connector configuration, all *new* folder creations will reflect the updated name.

**Existing folders:** Folders already created with old names (e.g., "FSC Corporate") will retain their old names. A data migration would be required to rename them retroactively, but this is a separate concern.

---

## Real-World Example: The E-deal Rebrand

### The Situation

E-deal was historically branded as "FSC Corporate" internally and in early customer-facing displays. It was later rebranded to "E-deal."

- **Integration ID:** Still "FSC Corporate" (cannot change without massive migration)
- **Display Name:** Changed to "E-deal"
- **Apsis One UI:** Now shows "E-deal" as the integration name
- **Apsis One Folders:** Previously showed "FSC Corporate" because they were created with the integration ID

### The Problem This PR Solves

Customers saw:
- "E-deal" in the integration catalog
- But "FSC Corporate" as the folder name in their workspace

This inconsistency is confusing and now avoided for future installations.

---

## Lead Entity Configuration Deep Dive

### Which Connectors Support Leads

[Erik Andersson]: Check the `installer.go` file for each connector's **additional entities** section:

**Support leads:**
- **Dynamics** (Microsoft Dynamics)
- **Tribe**
- **FSC Corporate** (Silhouette entity)
- Any others with an additional entity of type "lead"

**Do NOT support leads:**
- **Efficy Enterprise 12.0** (creates contacts only; no lead concept)
- **Efficy Enterprise 12.1** (creates contacts only; no lead concept)

### Why Leads Matter

Different CRM systems represent lead data differently. Some have a dedicated lead entity; others only have contacts. The generic connector framework allows us to map leads to the appropriate entity in each system.

[Erik Andersson]: When a form submission comes in, it becomes a lead in Apsis One. The lead can later be promoted to a full contact. We'll cover the details of this flow—including merging and profile lifecycle—in a future knowledge transfer session.

---

## Data Migration Considerations

### Current Status: E-deal Customers

[Erik Andersson]: E-deal currently has only 2-3 customers. A manual migration to rename existing folders from "FSC Corporate" to "E-deal" is feasible.

[Michal Rosikiewicz]: Rather than asking customers for account access, we should use an **internal delegated key** from Apsis One to run the migration.

### Access and Permissions

[Erik Andersson]: We are no longer permitted to request customer account access for administrative tasks. Instead, we should:

1. Use an **internal delegated key** (approved approach) from the Apsis One system
2. Run the migration script via API calls to update folder names
3. Avoid asking customers for credentials or account access

[Michal Rosikiewicz]: This is preferable to the old approach of requesting customer account tokens for one-off migration jobs. Even for a small number of accounts, we should follow the approved internal delegation pattern.

### Migration Script Planning

[Michal Rosikiewicz]: A migration script should:
- Extract the folder IDs from Apsis One using the internal delegated key
- Call the Apsis One API to rename folders
- Run on production accounts (after testing on staging/free accounts)

This is planned as a future story for the backlog, likely to be handled by Erik, Prem, and Łukasz.

---

## Code Navigation: Finding Connector Configurations

### Quick Reference

To understand how a connector works:

1. **Integration ID and Display Name:** `lib/connectors/[connector-name]/installer.go` → `init` → `StoreOption`
2. **Lead Support:** Same file → look for `additional entities` configuration
3. **Entity Mappings:** Same file → `entity_id`, `entity_name`, and unique identifier field
4. **Form Activities:** Check if the connector supports form submissions and how they're mapped

### Example: Checking Efficy Enterprise 12.1

```
File: lib/connectors/efficy-enterprise-12-1/installer.go

Integration ID: Efficy Enterprise 2 (internal; won't change)
Display Name: "12.1" (customer-visible; recently updated)
Lead Support: None (no additional entities configured)
Form Submissions: Supported—creates contacts directly (no intermediate lead stage)
```

### Example: Checking FSC Corporate

```
File: lib/connectors/fsc-corporate/installer.go

Integration ID: FSC Corporate (internal; permanent)
Display Name: "E-deal" (customer-visible; recently updated)
Lead Support: Yes (Silhouette entity configured as additional entity)
Form Submissions: Supported—creates contacts; silhouette records available for leads
```

---

## Integration IDs Remain Stable in System Logs and Contracts

[Erik Andersson]: Integration IDs appear in system logs and API responses, reflecting the permanent internal contract with the CRM system.

```
Example log entry:
integration_key: {account_id}:{integration_id}:{connector_logical_name}
```

The integration ID part (e.g., "FSC Corporate" for E-deal) never changes in logs or API contracts—this is why changing it would be so expensive. Customer-visible labels, however, always use the display name.

---

## Why This PR Matters for Future Maintainability

[Erik Andersson]: The key insight is that **display names are now the source of truth for customer-visible folder labels**. If a product owner decides to rebrand an integration in the future:

1. Change the display name in `installer.go`
2. All *new* folder creations will use the new name
3. No code changes needed elsewhere
4. Existing folders remain unchanged (can be migrated separately if needed)

This decouples the customer-visible branding (display name) from the internal contract (integration ID), making future changes much less risky.

---

## Key Takeaways

1. **Display names and integration IDs serve different purposes:** Integration IDs are immutable internal contracts; display names are mutable customer-visible labels.

2. **Folder creation now uses display names:** Future folder creations in Apsis One will reflect the current customer-visible name, not the internal integration ID.

3. **Lead entities are per-connector:** Check the `additional entities` section in each connector's `installer.go` to determine if leads are supported.

4. **Lead support varies widely:**
   - Efficy Enterprise 12.0 and 12.1 have no lead support
   - FSC Corporate, Dynamics, and Tribe have lead support
   - Maxo had a unique real-time subscription sync feature (now unsupported for new systems)

5. **Migration requires careful planning:** Use internal delegated keys to rename existing folders rather than requesting customer account access.

6. **Code navigation is straightforward:** All connector configuration lives in `lib/connectors/[name]/installer.go` under the `init` and `StoreOption` sections.

---

## Unresolved Questions & Action Items

### Completed
- ✅ PR approved (Erik approved the display names PR as a small, well-scoped change)

### Pending
- **Data migration for E-deal folder renames:** Create a backlog story to rename existing "FSC Corporate" folders to "E-deal" using internal delegated key + API calls. Owner: Michal Rosikiewicz (to create story); Implementation: Erik, Prem, and Łukasz.
- **Lead handling details:** Full knowledge transfer on lead creation, merging, and profile lifecycle is planned for a future session.

### Discussion Points for Later
- Sprint planning priorities for migration stories (Erik notes some customer issues may need to be prioritized before his and Prem's departure)