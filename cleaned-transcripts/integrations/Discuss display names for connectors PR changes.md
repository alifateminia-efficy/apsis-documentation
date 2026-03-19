---
source_file: Discuss display names for connectors PR changes.txt
domain: Integrations
topics: [Connector Configuration, Display Names vs Integration IDs, Folder Creation in Apsis, Lead Entity Handling, Additional Entities, PR Review]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Apsis One, Connector Configuration Files (installer.go, connector.go), Integration IDs, Display Names, Lead Attributes, Additional Entities, FSC Enterprise, FSC Corporate, E-Deal, Dynamics, Tribe, Maxo]
session_type: knowledge-transfer
---

## Session Overview

This session reviews a pull request that changes how display names are used in the Integrations domain when creating folders in Apsis One. The discussion covers the distinction between **integration IDs** (internal logical names used in API contracts) and **display names** (customer-facing labels), why this distinction matters, where connector configuration is stored, and the different reasons folders are created in Apsis (lead support and subscription sync). The session also touches on the need for a data migration strategy for existing customers.

---

## Display Names vs Integration IDs: Core Distinction

### The Problem Being Solved

When integrations are installed that support leads or real-time subscription sync, folders are created in Apsis One. Until this PR, these folders were named using the **integration ID** (the internal logical name), not the **display name** (what customers see). This creates a mismatch between what appears in the system and what is shown to customers.

### Why This Distinction Exists

**Integration IDs** are internal identifiers stored in connector configuration files. They are:
- Part of the agreed-upon protocol and API contract between the platform and the CRM system
- Extremely difficult to change because any change requires a massive data migration across all references in the system
- Not exposed to customers

**Display names** are the customer-facing labels for connectors. They:
- Can be easily changed in the system without database migrations
- Are what customers see in the UI
- Are what should appear in folder names since customers interact with those folders

[Erik Andersson]: > "Display names is very, very easy for us to change in the system, but the logical integration ID is on the contrary, very hard for us to modify in the system because one, the integration ID is part of the agreed upon protocol and API contract between us and the CRM system. Secondly, for us to change this, it would be like a massive data migration."

### Real-World Example: E-Deal (FSC Corporate)

The connector previously called "FSC Corporate" was renamed to "E-Deal" for customers. However:
- The internal integration ID remained `FSC_CORPORATE` (for API and data contracts)
- The display name was updated to "E-Deal"
- But the folder created in Apsis still showed the old internal name because the code used the integration ID, not the display name

---

## Connector Configuration: Where Display Names Live

### File Structure

Connector configuration is stored in the `lib/connectors` directory in connector-specific files:

```
lib/connectors/[connector-name]/
  installer.go (newer connectors)
  connector.go (older connectors like Dynamics)
```

### Configuration Section: `init` and `StoreOption`

Each connector config has an `init` and `StoreOption` section containing:
- **Integration ID**: The internal logical name (e.g., `FSC_ENTERPRISE_2`)
- **Display Name**: The customer-facing name (e.g., `FSC Enterprise 2`)

[Erik Andersson]: "In here we have the integration ID. So I'm going to use FSC Enterprise 2 here now, like the generic connector version of FSC Enterprise. So you see here the integration ID for this is FSC Enterprise 2. That's what we have called it internally just to have something to name it, but then you have a display name also which up until now has been FSC Enterprise too as well."

---

## When and Why Folders Are Created in Apsis One

### Case 1: Lead Support

When an integration that supports lead handling is installed, the platform creates a **lead ID attribute folder** in Apsis One.

**Rationale**: Most CRM systems support contacts/persons as their main customer entity, but some also support leads as a separate entity type. The platform accommodates this by:
- Creating a dedicated lead attribute/folder when a lead entity is configured
- Allowing leads and contacts to be handled separately
- Supporting future lead-to-contact conversion (covered in future sessions)

Connectors with lead support include:
- FSC Corporate (has a "Silhouette" entity representing leads)
- Dynamics (newer versions)
- Tribe

Connectors without lead support:
- FSC Enterprise (no lead concept; submits forms as pure contacts)
- FSC Enterprise 2 (no lead concept)

### Case 2: Real-Time Subscription Sync

Only **Maxo** implemented this flow. The system:
- Listens for webhooks when Maxo customers create consent lists
- Creates a **subscription folder** under the integration for each consent type
- Example: If a customer created a consent list called "technical newsletters," a folder would be created with that name

[Erik Andersson]: "For Maxo we implemented a system where we had like essentially real time sync for subscriptions. So like not the consent, but what they actually had consent in. So if they created a consent list called like technical newsletters, they sent a web hook to us and we would create a subscription folder under the integration ID name."

This flow is no longer active, so subscription folders will not appear for new installations.

---

## Additional Entities Configuration

Beyond the main entity (contacts/persons), connectors can support **additional entities** such as leads.

### How to Find Additional Entity Configuration in Code

Look in the connector's configuration file for the `additional entities` or `additional_entities` section:

```
[Additional Entities Configuration]
- Entity ID (what is it called in the CRM?)
- Entity Name (internal representation)
- Unique Identifier Field (which field maps to the profile ID?)
- Supported Operations (what can be done with this entity?)
```

[Erik Andersson]: "You can see which which integrations like code wise you can see it if they have any additional entities configured like you will often see like a lead entity here like what is the ID for that entity? What is the name of it? What field on the profile does it use as the unique identifier and whatever you can do with it?"

### Example: FSC Corporate Configuration

FSC Corporate includes an additional entity for leads:

```
Main Entity: Contacts
Additional Entities:
  - Lead (Silhouette entity)
```

When a customer installs FSC Corporate, they will see both a standard contacts folder and an additional leads folder.

---

## The PR Changes: Display Name Usage

### What Changed

The PR modifies where the platform creates folders to use **display names** instead of **integration IDs**:

```
Old behavior: folder name = integration_id
New behavior: folder name = display_name
```

### Examples of Changes

| Connector | Old Folder Name | New Folder Name |
|-----------|-----------------|-----------------|
| FSC Corporate | FSC_CORPORATE | E-Deal |
| FSC Enterprise 2 | FSC_ENTERPRISE_2 | FSC Enterprise 12.1 |
| FSC Enterprise | FSC_ENTERPRISE | FSC Enterprise 12.0 |

### Important Constraints

This change applies **only to new folder creations**:
- Existing folders created with old names will retain those names
- Only when new integrations are installed or new entities are added will the new display names be used
- No automatic retroactive renaming of existing folders

[Erik Andersson]: "So the whole point of this PR is that the display name should be utilized when we create the folder and not the integration ID because this is actually something which the customer can see. And you can see here at the same time here now we have actually made the change. So now in the future when you install a folder we will create it with the actual the actual display name, so it is changed in the future if someone decides that they don't know where we should re-change it from E deal to, I don't know something else. Then you only need to change the display name in the on the integration configuration and then this will be proliferated to any other service which reference it."

### Why This Matters for Future Changes

If a display name needs to be changed later (e.g., from "E-Deal" to something else):
- Only the display name in the connector configuration needs to be updated
- All new folders will automatically use the new name
- This cascades to any other service referencing the display name
- Migration of existing folders remains a separate concern

---

## Data Migration Strategy for Existing Folders

### Current Status

Only **E-Deal (FSC Corporate)** customers are affected by the need to rename existing folders. Current customer count:
- Approximately 2-3 E-Deal customers with the old "FSC_CORPORATE" folder names

### Migration Options Discussed

#### Rejected Approach: Direct Account Access
[Michal Rosikiewicz]: > "I think it's a bad approach because we ask clients to give the access to the account just to have one job that we'll use to update the name of the folder."

The approach of asking customers for account access to run a one-time migration script was rejected as poor practice.

#### Preferred Approach: Internal Delegated Keys
[Michal Rosikiewicz]: "We can have internal delegated key from IO and we'll be able to do this migration with... we create some... yeah, yeah."

The team prefers using internal delegated API keys from the IO service, which would allow:
- Running the migration without requesting customer account access
- Using an approved, internal authentication mechanism
- A self-service migration script approach similar to what's used in other systems (AO)

### Implementation Notes

- A story/task needs to be created to implement the migration
- The migration should extract folder IDs from affected customer accounts
- A script using delegated keys would execute the rename operation
- Access provisioning: SOC may need to grant permissions to the integration service

[Erik Andersson]: > "Usually for these CRM customers that we have like someone to collaborate with, we have essentially a standing consent to do whatever we want. We just need to ask SOC to give us access to it."

---

## Integration IDs in System Logs and APIs

### Where Integration IDs Appear

Integration IDs are still used throughout the system and appear in:
- Log entries (integration key format includes the logical name)
- API contracts and responses
- Internal data models

### Log Format Example

Logs show the integration key as:

```
[account-section]_[logical-name]
Example: ACCOUNT123_FSC_ENTERPRISE_2
```

[Erik Andersson]: "For the integration key, like here you have the integration ID. So if it was for FSA Enterprise 12.1 it would say FSA Enterprise 2 because that is our logical name... But again this is not exposed to the customer whereas this folder is exposed to the customer and then we need to utilize the customer known name for it."

### Why This Distinction Matters

Since integration IDs are:
- Internal facing only
- Not exposed to customers
- Required for API compatibility

They remain unchanged despite display name updates. The folder name change is specifically about customer-facing artifacts.

---

## Translation and Localization Implications

[Erik Andersson]: "This is of course also relevant for any potential translation cases, et cetera."

By using display names for folder creation, the system enables:
- Future localization of folder names for different regions/languages
- Easier display name management for translation purposes
- Display names can be changed without affecting internal system references (unlike integration IDs)

---

## Related Future Discussions

### Lead Entity Management (To Be Covered)

The session identified additional lead-related topics for future knowledge transfer sessions:

[Erik Andersson]: "Primarily how, primarily how they are gathered, how this works together with the outbound mappings and like how do we handle the merging like when a lead becomes like a fully-fledged contact. Like how do we keep track of those of the lead profile and the contact profile in Apsis?"

Key areas to cover:
- How leads are gathered from CRM systems
- Integration of lead gathering with outbound mappings
- Lead-to-contact conversion process
- Profile merging: tracking lead and contact profiles in Apsis when a lead converts to a contact

---

## PR Approval Status

[Erik Andersson]: "In contrast to the other PR which I said we shouldn't merge, I have already gone ahead and approved this one because it was quite small, but I'm happy that you looked at it."

The display names PR has been **approved and ready for merge**. This is a low-risk change given:
- Small scope of changes
- No retroactive impact on existing data
- Backwards compatible
- Clear improvement to customer experience

---

## Key Takeaways

1. **Display names are for customers, integration IDs are internal**: Use display names for anything customer-facing (like folder names); integration IDs remain as internal contracts that are difficult to change.

2. **Folder creation now uses display names**: New folders created for lead support or other entities will use customer-facing display names, improving clarity and enabling easier localization.

3. **Configuration location matters**: Connector display names and integration IDs are configured in `lib/connectors/[connector]/installer.go` or `connector.go` in the `init` and `StoreOption` sections.

4. **Additional entities are how leads (and other non-standard entities) are handled**: Check the `additional entities` section of a connector's config to see what extra entities it supports beyond the main contact/person entity.

5. **Lead support varies by connector**: Only connectors with an additional lead entity will create lead folders. FSC Corporate, Dynamics, and Tribe support leads; FSC Enterprise and FSC Enterprise 2 do not.

6. **Migration needed for E-Deal**: The rename of FSC Corporate to E-Deal requires a migration of existing folder names. This should use internal delegated keys rather than requesting customer access.

7. **Future changes to display names are now easier**: If a display name changes in the future, new folders will automatically use the new name, though existing folder names will remain unchanged.

---

## Unresolved Questions & Action Items

### Action Items

- **Michal Rosikiewicz**: Create a story for the E-Deal folder name migration using internal delegated keys from IO service
- **Backlog prioritization**: Discuss with Prem and Lukasz which stories should be prioritized, especially those addressing known customer issues or blocking problems before team members leave

### Open Questions

- What is the priority for the E-Deal migration story relative to other backlog items?
- How will the internal delegated key approach integrate with the broader IO service authentication system?