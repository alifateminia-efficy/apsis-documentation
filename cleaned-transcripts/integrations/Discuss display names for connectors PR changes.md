---
source_file: Discuss display names for connectors PR changes.txt
domain: Integrations
topics: [Display Names vs Integration IDs, Connector Configuration, Folder Creation in Apsis, CRM Integration Patterns, Lead Entity Handling]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Apsis One, CRM Connectors, Integration Platform, Lib Connectors, Installer Configuration Files, Lead Attributes, Subscription Folders]
session_type: architecture-review
---

## Session Overview

This session reviewed a pull request that changes the connector folder creation logic in Apsis One to use **display names** instead of **integration IDs** when creating folders for CRM integrations. Erik Andersson explained the distinction between logical integration IDs (internal, part of API contracts) and display names (customer-facing, easily changeable), and walked through two primary use cases where folders are created: lead entity handling and real-time subscription syncing. The discussion covered configuration locations, the implications of this change, and the rationale for this approach to maintain customer-facing consistency while preserving internal API stability.

---

## Display Names vs. Integration IDs: The Core Problem

### The Distinction

**Integration IDs** are internal logical names used to identify connectors within the system. **Display names** are customer-visible labels that can differ from the integration ID. For example:

- A connector with integration ID `FSC_Enterprise_2` might have display name `FSC Enterprise 2`
- The same connector might have its display name changed to `12.0` or `12.1` for customer visibility
- Historical example: EDeal was previously called FSC Corporate internally but renamed to EDeal for customers

### Why This Matters

[Erik Andersson]: The problem is that display names and logical names can diverge. Display names are very easy to change in the system—it's essentially a UI label update. Integration IDs are extremely difficult to change because:

1. **API Contract**: The integration ID is part of the agreed-upon protocol and API contract between the platform and the CRM system
2. **Data Migration Complexity**: Changing an integration ID requires massive data migration everywhere the ID is referenced in the database
3. **Backwards Compatibility**: Existing integrations, logs, and historical data all reference the original ID

Changing a display name is fast; changing an integration ID is a breaking change requiring data migration.

---

## Connector Configuration Structure

### Location of Configuration Files

Configuration for each connector is defined in the **lib/connectors** directory. The primary file is typically:
- `installer.go` for most connectors
- For older connectors like Dynamics, configuration may be in different locations
- For newer connectors, it may be `connector.go`

### Configuration File Structure

Within the installer configuration, there is an `init` and `store option` section that contains:

```
Integration ID: FSC_Enterprise_2
Display Name: FSC Enterprise 2
```

This configuration section defines both the internal identifier and the customer-visible name.

---

## Two Primary Use Cases for Folder Creation in Apsis One

### Case 1: Lead Entity Folders

When installing an integration that supports **lead handling**, the platform creates a dedicated lead ID attribute folder in Apsis One.

**Background**: By default, most CRM systems work with the main customer entity (contacts or persons). However, many CRMs also support a separate lead entity for lead gathering. The platform bridges this by creating a dedicated folder structure.

[Erik Andersson]: When we install a folder or integration that supports leads, we create a lead ID attribute inside Apsis. The integrations by default support contacts or persons—the main customer entity in the CRM system—but we also support lead gathering. For that we create a dedicated lead attributes by creating a dedicated folder. Previously we used the integration ID instead of the display name.

**Which connectors support leads**: 
- FSC Enterprise variants: FSC Corporate (EDeal) has the `Silhouette` lead entity
- Dynamics: Supports lead entities in newer versions
- Tribe: Supports lead entities
- Maxo: Does not have a traditional lead entity folder (uses different approach for subscriptions)
- FSC Enterprise 12.0 and 12.1: Do NOT support leads—they only work with contacts

**Why configuration differs by connector**: You can identify lead support in code by checking the `additional_entities` configuration section in the installer file. For example, FSC Corporate shows:

```
additional_entities:
  - entity_name: Silhouette
    entity_id: [ID value]
    unique_identifier_field: [field name]
```

### Case 2: Real-Time Subscription Syncing (Maxo)

[Erik Andersson]: For Maxo we implemented a system where we had real-time sync for subscriptions—not the consent itself, but what they actually had consent in. If they created a consent list called "technical newsletters", they sent a webhook to us and we would create a subscription folder under the integration ID name.

**Example scenario** (would apply if EDeal supported this flow):
- A folder would be created named after the integration ID (e.g., `FSC_Corporate`)
- Consent lists from the CRM would appear as subfolders
- Real-time updates via webhook would sync consent status

**Current status**: Only Maxo currently supports this subscription sync flow. This folder will no longer appear for any new installations of other systems using this approach.

---

## The PR Change: Using Display Names Instead of Integration IDs

### What Was Changed

The pull request modifies folder creation logic to use the **display name** from the connector configuration instead of the **integration ID**.

**Examples from the PR**:
- `FSC_Corporate` → `EDeal` (display name change reflected in folder name)
- `FSC_Enterprise_2` → `12.1` (display name change reflected in folder name)
- `FSC_Enterprise` → `12.0` (display name change reflected in folder name)

### Code Changes

The configuration update changed from:

```
Integration ID: FSC_Enterprise_2
```

to:

```
Display Name: FSC Enterprise 2
```

for folder naming purposes.

### Why This Matters to Customers

[Erik Andersson]: The folder names in Apsis are something which the customer can see. The whole point of this PR is that the display name should be utilized when we create the folder and not the integration ID, because this is actually something which the customer can see.

Customers see the folder names in their Apsis One interface. If a folder was created as `FSC_Corporate` but the system now calls it `EDeal`, customers see a mismatch between their configuration and what appears in the platform.

### Future Impact

Going forward, if a display name is changed:

1. **New folders** created after the change will use the new display name
2. **Existing folders** retain their old names (would require separate data migration)
3. **Log entries and API references** still use the integration ID (internal, not customer-facing)
4. **No change to integration ID**: The internal logical name remains unchanged, preserving API contracts and avoiding data migration

[Erik Andersson]: So the whole point of this PR is that in the future when you install a folder, we will create it with the actual display name. So it is changed in the future if someone decides that we should re-change it from EDeal to something else. Then you only need to change the display name in the integration configuration and then this will be propagated to any other service which references it. Of course, already-created folders will still have the old name—that would require its own data migration—but we would not create new folders with the old incorrect name.

---

## Internal vs. Customer-Facing References

### Where Integration IDs Remain (Internal Use)

Integration IDs continue to be used in:
- **Log entries and monitoring**: When you see `FSC_Enterprise_2` in logs, this is the internal identifier
- **API responses**: The integration key contains the account section and logical connector name
- **Database references**: All internal data structures continue to reference the integration ID
- **API contracts**: External CRM communication uses the original integration ID

### Where Display Names Appear (Customer-Facing)

Display names are used in:
- **Folder names in Apsis One**: What customers see in their UI
- **Connector list page**: Customer-visible integration options
- **Documentation and user-facing labels**: Translation, localization, marketing

[Erik Andersson]: This is not exposed to the customer whereas this folder is exposed to the customer, and then we need to utilize the customer-known name for it. And this is of course also relevant for any potential translation cases.

---

## Configuration Discovery: Finding Additional Entities in Code

To determine which integrations support leads or other additional entities, check the connector's installer file for the `additional_entities` section:

```
additional_entities:
  - name: "Lead Entity Name"
    field_for_unique_id: [identifier field]
```

If this section is absent or empty, the connector only supports the main entity (contacts).

[Michal Rosikiewicz]: So the key is additional_entities.

[Erik Andersson]: Yep. If you install Dynamics, Sideshop, FSC Enterprise 2, or Tribe, you will see this additional folder. But only Maxo supported the flow with the real-time subscription sync. So this subscription folder will not appear anymore for any existing systems.

---

## Migration Strategy for Existing Folders

### Current Situation

EDeal (formerly FSC Corporate) currently has only 2-3 active customers. Their existing folder names still reflect the old internal name.

### Migration Approach Discussion

[Erik Andersson]: I haven't discussed yet with Emmanuel, who is the product owner for EDeal. It might very well be that we will do a migration because we don't have that many EDeal customers. We have two or three right now, so we can just do that with manual API calls to Apsis. Most likely we don't need to sit and do any heavy migration scripts.

[Michal Rosikiewicz]: If it's like a simple call to Apsis, then we can have a script actually that will take... We can run a migration. Two kind of offends on AT transfer stuff for free accounts.

### Access and Permission Considerations

**Current approved approach**: Use internal delegated keys from IAM rather than asking customers for account access.

[Michal Rosikiewicz]: I think it's a bad approach because we ask clients to give access to the account just to have one job that we'll use to update the name of the folder. For me, it's something we should avoid.

**Rationale**: 
- Customers should not need to grant account access simply to rename a folder
- Internal delegated keys from IAM are the preferred method for such maintenance operations
- This is more secure and reduces customer friction

[Erik Andersson]: If we have an approved approach where we use internal keys, I am very happy to do that instead.

---

## Technical Approval and Status

[Erik Andersson]: I have already gone ahead and approved this PR because it was quite small. In contrast to the other PR which I said we shouldn't merge, I have already approved this one.

The change is considered **low-risk** because:
1. It only affects folder creation logic going forward
2. It doesn't modify integration IDs or API contracts
3. Existing folders are not affected
4. The configuration change is minimal

---

## Related Topics for Future Discussion

The session identified that a deeper knowledge transfer session will cover:

- **Lead handling in detail**: How leads are gathered from CRM systems
- **Lead-to-contact conversion**: How the system handles when a lead becomes a fully-fledged contact
- **Outbound mappings**: How lead data maps to Apsis attributes
- **Profile merging**: How the platform tracks and manages both lead and contact profiles for the same entity

[Erik Andersson]: Primarily how they are gathered, how this works together with the outbound mappings, and like how do we handle the merging—like when a lead becomes like a fully-fledged contact. How do we keep track of the lead profile and the contact profile in Apsis? We'll touch base on that then.

---

## Key Takeaways

1. **Display Names vs. Integration IDs**: Display names are customer-visible and easily changeable; integration IDs are internal identifiers that are difficult to change due to API contracts and data migration complexity.

2. **Folder Creation Changed**: The PR changes folder creation in Apsis One to use display names instead of integration IDs, ensuring customer-facing folder names match the connector's current customer-visible name.

3. **Two Main Folder Use Cases**: 
   - Lead entity folders for CRMs that support separate lead entities
   - Subscription sync folders (currently only Maxo)

4. **Configuration Location**: Connector configuration is in `lib/connectors/[connector]/installer.go` with an `init` and `store option` section containing both Integration ID and Display Name.

5. **Finding Lead Support**: Check the `additional_entities` section in a connector's installer file to determine what lead or other entity support it has.

6. **Backward Compatibility Preserved**: Existing folders keep old names; only new folders use new display names. Integration IDs remain unchanged in logs and APIs.

7. **Internal vs. External**: Integration IDs continue to be used in all internal references (logs, APIs, contracts); display names appear only in customer-facing UI elements.

8. **Migration Approach**: For existing customer folders with old names, use internal delegated keys from IAM for access rather than asking customers for account credentials.

---

## Action Items

- **Michal Rosikiewicz**: Create a story for migrating existing EDeal folder names using internal delegated keys approach
- **Erik Andersson**: Discuss migration plan with Emmanuel (EDeal product owner) to determine if/when to execute folder name migration
- **Future**: Plan deeper knowledge transfer session on lead handling, lead-to-contact conversion, and outbound mappings

---

## Unresolved Questions

- Will a migration be executed for the 2-3 existing EDeal customers, and if so, on what timeline?
- Specific implementation details of the internal delegated key approach for Apsis access (this was being discussed as new policy)