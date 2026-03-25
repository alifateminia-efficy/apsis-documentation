---
source_file: Discuss display names for connectors PR changes.txt
domain: Apsis One Integrations
topics: [Display Names vs Integration IDs, Folder Creation in Apsis One, Connector Configuration, Lead Entity Handling, Data Migration Strategy]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Apsis One, Integration Platform, Generic Connector, Connector Configuration Files (installer.go), Lead Attributes, Additional Entities, E-Deal Integration, Efficy Enterprise 12.0, Efficy Enterprise 12.1, FSC Corporate]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Lead creation, E-deal (Efficy Corporate)]
---

## Session Overview

This knowledge transfer session covers a PR (pull request) that changes how folder names are created in Apsis One when integrations are installed. The key change is using the **display name** of an integration (user-facing, changeable) instead of the **integration ID** (internal, stable identifier) when creating folders. The discussion explores why this distinction matters, where folder creation happens in the codebase, and how this affects different connector types—particularly those supporting lead entities. The session also touches on a potential data migration for existing E-Deal customers.

---

## Display Names vs. Integration IDs: The Core Problem

### Why the Distinction Matters

[Erik Andersson]: The fundamental issue is that **display names** and **integration IDs** can differ significantly. The display name is what customers see in the UI and what is visible to end users, while the integration ID is the internal logical identifier used by the platform.

For example, **E-Deal** was previously called **FSC Corporate** internally. The integration ID `FSC_Corporate` cannot be easily changed because:

1. The integration ID is part of the **agreed-upon protocol and API contract** between Apsis One and the CRM system
2. Changing the integration ID would require a **massive data migration** everywhere the ID is referenced in the system
3. Changing the display name, by contrast, is trivial—it only affects the UI label

[Erik Andersson]: > "Display names is very, very easy for us to change in the system, but the logical integration ID is On the contrary, very hard for us to modify in the system."

### Real-World Example: Efficy Enterprise Naming

The Efficy Enterprise connectors illustrate this:
- **Integration IDs** (internal): `Efficy_Enterprise`, `Efficy_Enterprise_2`
- **Display Names** (customer-facing): `Efficy Enterprise 12.0`, `Efficy Enterprise 12.1`

Customers see `12.0` and `12.1`, but internally the system still uses `Efficy_Enterprise` and `Efficy_Enterprise_2`.

---

## Where Folder Creation Happens: Two Key Scenarios

[Erik Andersson]: Folders are created in Apsis One in two specific scenarios within the integration platform:

### Scenario 1: Lead Entity Support

When an integration that supports **leads** is installed, Apsis One creates a dedicated **lead ID attribute** folder.

The rationale: integrations typically support contacts or persons (the main customer entity in the CRM), but many also support lead gathering as a separate entity type. To handle this, a dedicated folder is created for leads.

**Problem (before this PR)**: The folder was named using the integration ID instead of the display name.

### Scenario 2: Subscription/Consent Sync (Maxo Real-Time Example)

[Erik Andersson]: For systems like Maxo that supported real-time subscription synchronization, when a customer created a consent list (e.g., "technical newsletters"), Maxo would send a webhook and Apsis would create a **subscription folder** under the integration ID name.

For example, if E-Deal (formerly FSC Corporate) had supported this flow, a folder would have been created called `FSC_Corporate` under subscriptions. Instead, it should have been named `E-Deal` (the display name).

**Note**: [Erik Andersson] clarified that Maxo was the only integration that actually supported this real-time subscription sync flow; this pattern doesn't exist for other integrations currently.

---

## Connector Configuration: Where Display Names Are Defined

### Location: `installer.go` and Connector Config Files

[Erik Andersson]: In the `/lib/connectors/` directory, each connector has a configuration file (usually `installer.go`; for very old connectors like Dynamics, it may be different). Within this file, there is an `InitStoreOption` section containing both:

1. **Integration ID** – the internal logical identifier
2. **Display Name** – the customer-facing name

Example structure for FSC Enterprise 2 (generic connector version):
```
Integration ID: FSC_Enterprise_2
Display Name: FSC Enterprise 2 (can be changed to "E-Deal", "Efficy Enterprise 12.1", etc.)
```

### Why This Configuration Matters

The display name is mutable and can be updated at any time to reflect rebranding or naming changes. When the PR is merged, any folder creation that happens *after the configuration change* will use the new display name.

[Erik Andersson]: > "If someone decides that they don't know where we should re-change it from E-Deal to something else, then you only need to change the display name in the integration configuration and then this will be propagated to any other service which references it."

---

## Impact of the PR: What Changed

### The Change

The PR modifies the folder creation logic to use the **display name** instead of the **integration ID** when folders are created in Apsis One.

### Examples of Display Name Changes in the PR

- `FSC_Corporate` → `E-Deal`
- `Efficy_Enterprise_2` → `Efficy Enterprise 12.1`
- `Efficy_Enterprise` → `Efficy Enterprise 12.0`

### What Did NOT Change

**Important**: The integration IDs themselves remain unchanged. Logs, API contracts, and internal identifiers continue to use the integration ID. Only the folder names visible to customers are affected.

[Erik Andersson]: > "If we check in the logs here for example, for the integration key, like here you have the integration ID. So if it was for Efficy Enterprise 12.1 it would say Efficy Enterprise 2 because that is our logical name... This is not exposed to the customer whereas this folder is exposed to the customer and then we need to utilize the customer known name for it."

### Data Migration of Existing Folders

[Erik Andersson]: Existing folders created with the old (integration ID) names will keep those names—they are not automatically renamed. Only folders created *after* the PR is merged will use the new display names.

---

## Lead Entity Configuration: Understanding Additional Entities

[Michal Rosikiewicz]: When viewing profiles from connectors like Efficy Enterprise 12.1, some users may not see a dedicated lead folder.

[Erik Andersson]: This is expected behavior. Whether a lead folder is created depends on the connector's configuration of **additional entities**.

### How Additional Entities Work

In the connector configuration (`installer.go`), there is a primary entity (contacts, persons, etc.) and any **additional entities** configured on top of it. Lead entities are stored in the `additional_entities` section.

Example from FSC Corporate:
```
Additional Entities: [Lead Entity Configuration]
Entity ID: <their_representation_of_leads>
Field used as unique identifier: <specific_field>
```

### Which Connectors Support Leads

- **FSC Corporate** – has a lead entity (called "Silhouette" in their system)
- **Dynamics** – supports leads
- **Tribe** – supports leads
- **Efficy Enterprise 12.0** – does NOT support leads (they only create contacts)
- **Efficy Enterprise 12.1** – does NOT support leads
- **Maxo** – supported leads (in addition to the subscription sync feature)

[Erik Andersson]: > "If you were to install Dynamics, FSC Corporate, or Tribe, you will see this additional folder. But only Maxo supported the flow with the real time subscription sync here. So this folder will not appear anymore for any existing Maxo systems."

### How to Identify Lead Support in Code

To determine if a connector supports leads, check for `additional_entities` configuration in the connector's `installer.go`. The presence and configuration of these entities determines what folders are created during installation.

---

## Data Migration Strategy for Existing Customers

### The E-Deal Situation

Currently, Apsis One has approximately **2-3 E-Deal customers** with existing folders named `FSC_Corporate` (the old integration ID). The PR does not retroactively rename these existing folders.

### Proposed Migration Approach

[Michal Rosikiewicz] and [Erik Andersson] discussed two approaches:

#### Approach 1: Customer Account Access (Previously Used, Now Restricted)

- Request access to customer accounts
- Use actual API tokens for those accounts
- Call the Apsis API directly to rename folders

**Problem**: This requires asking customers for account access, which Michal Rosikiewicz views as suboptimal compliance practice.

[Michal Rosikiewicz]: > "For me, it's something we should avoid. Even if it's still only for three accounts."

#### Approach 2: Internal Delegated Keys (Preferred)

[Michal Rosikiewicz]: Instead of requesting customer account access, use **internal delegated keys** from the authentication/identity system.

- Create an internal delegated key with limited permissions
- Use this key to run the migration script
- Run migration on production accounts without requiring customer involvement

[Michal Rosikiewicz]: > "We can run it on production on the other hand. We can have internal delegated keys from IO and we'll be able to do this migration."

### Current Status

- No migration has been scheduled yet
- Erik Andersson hasn't discussed this with Emmanuel (the product owner for E-Deal)
- A story/ticket will be created to track this work
- The approach and priority will be determined in backlog planning sessions

[Erik Andersson]: > "It might very well be that we will do a migration because we don't have that many E-Deal customers, I think. We have two or three right now, so we can just do that with manual API calls to Apsis."

---

## Access and Permissions for Data Migrations

[Erik Andersson]: For CRM customers with standing business relationships, there is typically **standing consent to access their accounts** if needed. However, the process requires coordination:

1. Request access through **SOC** (Security Operations Center) to get account credentials
2. Use official API endpoints to perform the migration

[Erik Andersson]: > "Usually for these CRM customers that we have, someone to collaborate with, we have essentially a standing consent to do whatever we want. We just need to ask SOC to give us access to it."

**Policy Change**: The team is no longer allowed to use system-level keys directly. All account access must go through proper permission channels.

---

## PR Approval and Code Review Notes

[Erik Andersson]: The PR in question was **already approved** by Erik Andersson because:

1. The change is small and focused
2. It clearly separates display names (user-facing) from integration IDs (internal)
3. The rationale is sound: folders visible to customers should use customer-facing names
4. It enables future rebranding changes with simple configuration updates

---

## Key Takeaways

1. **Display names and integration IDs serve different purposes**: display names are customer-facing and changeable; integration IDs are internal, stable, and tied to API contracts.

2. **Folder creation happens in two scenarios**: when installing integrations with lead support, and for real-time subscription syncs (Maxo only). Both scenarios now use display names instead of integration IDs.

3. **Connector configuration is centralized**: both display names and integration IDs are defined in `installer.go` under the `InitStoreOption` section.

4. **Future changes are simplified**: if a connector needs rebranding, only the display name needs to be changed in the configuration. New folder creation will automatically reflect this change.

5. **Lead entity support varies by connector**: check the `additional_entities` configuration to determine which connectors support leads (e.g., FSC Corporate, Dynamics, Tribe support leads; Efficy Enterprise 12.0/12.1 do not).

6. **Data migration for existing folders requires planning**: existing folders with old names will need a separate migration. The team is considering using internal delegated keys to avoid requesting customer account access.

7. **Access policies have changed**: system-level keys are no longer used. All data migrations requiring customer account access must go through proper authorization channels (SOC).

---

## Unresolved Questions & Action Items

1. **Migration Story Creation**: [Michal Rosikiewicz] committed to creating a story/ticket to track the E-Deal folder name migration for existing customers.

2. **Migration Approach Finalization**: The team needs to confirm the internal delegated key approach with the appropriate stakeholders before implementation.

3. **Prioritization**: Migration priority will be determined in upcoming backlog planning sessions, along with other customer-impacting issues that need resolution before team transitions.

4. **Lead Entity Deep Dive**: A follow-up knowledge transfer session will cover lead entity handling in detail, including:
   - How leads are gathered
   - Integration with outbound mappings
   - How leads are merged when they become full contacts
   - Profile tracking in Apsis (lead vs. contact)