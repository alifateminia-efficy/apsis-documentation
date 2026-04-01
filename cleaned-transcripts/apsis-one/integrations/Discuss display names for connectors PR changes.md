---
source_file: "Discuss display names for connectors PR changes.txt"
domain: Apsis One Integrations
topics: [connector display names, integration IDs, folder creation in Apsis One, lead entity configuration, additional entities, folder name migration, access control for migrations]
speakers: ["Erik Andersson (senior/outgoing developer)", "Michal Rosikiewicz (incoming developer)"]
key_components: [Apsis One, integration platform, connectors lib, installer.go, FSC Corporate/E Deal, FSC Enterprise 12.0, FSC Enterprise 12.1 (FSC Enterprise 2), Dynamics, Tribe, Maxo, Audience API]
session_type: knowledge-transfer
---

# Display Names for Connectors — PR Review & Knowledge Transfer

## Session Overview

Erik Andersson walks Michal Rosikiewicz through a PR that changes how folder names are generated when installing integrations in Apsis One. The core issue is that folder names were previously derived from internal **integration IDs** (logical names) rather than human-readable **display names**, causing customer-visible folders to show internal identifiers. The session covers where this configuration lives in the codebase, which integrations are affected, how the lead entity folder and Maxo subscription folder creation work, and a discussion about how to migrate existing incorrectly-named folders for current E Deal customers.

---

## Background: The Two Types of Connector Names

Every connector in the system has two distinct name fields, defined in its configuration file:

1. **Integration ID** (logical name) — the internal identifier used in the API contract with the CRM system, in logs, and throughout the database. This is part of the agreed-upon protocol between Apsis and the CRM vendor.
2. **Display name** — the human-readable label shown in the UI and, critically, used as the folder name in Apsis One Audience.

### Why They Differ and Why This Matters

Display names are trivial to change. Integration IDs, by contrast, are extremely difficult to change because:
- They form part of the API contract with the CRM system.
- Changing them would require a massive data migration across every reference in the system.

**Example:** The connector internally known as `fsc-corporate` was eventually rebranded to **E Deal**. The integration ID `fsc-corporate` remains unchanged, but the display name was updated to `E Deal`. Before this PR, folder names in Apsis One Audience reflected the integration ID (`fsc-corporate`), not the display name — meaning customers saw internal identifiers rather than the product name they knew.

Similarly, what customers know as **FC Enterprise 12.0** and **FC Enterprise 12.1** are internally `fsc-enterprise` and `fsc-enterprise-2` respectively.

---

## Where Connector Configuration Lives in the Codebase

[Erik Andersson]: The configuration is inside the `lib` folder in the connectors repo. Each connector has a configuration file — naming conventions vary by age:

```
lib/connectors/<connector-name>/installer.go   # newer connectors
lib/connectors/<connector-name>/connector.go   # oldest connectors (e.g., Dynamics)
```

Within that file, there is an `init` / `store options` section that contains both the `integration ID` and the `display name`. Prior to this PR, the display name for several connectors was simply set to the same string as the integration ID, which is where the problem originated.

**Example from FSC Enterprise 2 config (before PR):**
- Integration ID: `fsc-enterprise-2`
- Display name (old): `fsc-enterprise-2` ← incorrect, same as logical name
- Display name (new): `FC Enterprise 12.1` ← customer-facing name

---

## When Folders Are Created in Apsis One

There are two scenarios in which the integration platform creates folders in Apsis One Audience:

### 1. Lead Entity Folder (on integration install)

When an integration that supports **lead handling** is installed, a dedicated **lead folder** is created in Apsis One Audience under that integration. This is separate from the main contact/person entity that all integrations support by default.

Not all integrations have this. The determining factor is whether an `additional entities` configuration is present in `installer.go`. Specifically, look for a **lead entity block** that defines:
- The entity ID
- The entity name
- The field used as the unique identifier on the profile
- Supported operations

**Integrations that have a lead entity (and therefore create a lead folder):**
- FSC Corporate / E Deal — uses an entity called `silhouette` as their lead representation
- Dynamics (new version, "Sideshop")
- Tribe

**Integrations that do NOT have a lead entity:**
- FSC Enterprise 12.0 (`fsc-enterprise`) — creates pure contacts directly; no lead concept exists in their CRM flow. Forms submitted via Apsis go straight to the contact entity.

[Erik Andersson]: > "You can always check code-wise whether an integration has additional entities configured — you will often see a lead entity block with what the ID for that entity is, what the name of it is, what field on the profile it uses as the unique identifier."

### 2. Maxo Real-Time Subscription Sync Folder

**Maxo** had a dedicated mechanism for real-time subscription sync — not consent itself, but the consent *lists* customers subscribed to. When a customer created a consent list (e.g., "Technical Newsletters") in Maxo, Maxo sent a webhook to the integration platform, which then created a **subscription folder** under the integration name in Apsis One.

> This folder creation path used the integration ID as the folder name — the same bug addressed by this PR. **Note:** This Maxo subscription sync flow no longer applies to any existing systems — no new folders of this type will be created going forward.

---

## The PR Change: Using Display Name Instead of Integration ID

The fix is straightforward: in the folder-creation logic, the code was changed to use the **display name** from the connector options rather than the integration ID.

**Folder name changes applied:**
| Internal Integration ID | Old folder name (incorrect) | New folder name (correct) |
|---|---|---|
| `fsc-corporate` | `fsc-corporate` | `E Deal` |
| `fsc-enterprise-2` | `fsc-enterprise-2` | `FC Enterprise 12.1` |
| `fsc-enterprise` | `fsc-enterprise` | `FC Enterprise 12.0` |

### What This Does NOT Change

The integration ID itself is untouched. In logs, the **integration key** (the "holy trinity" of account section + logical connector name + account identifier) still uses the internal integration ID. This is internal and never customer-facing.

### Forward Compatibility

Going forward, if a product owner decides to rename a connector (e.g., renaming E Deal to something else), the team only needs to update the display name in `installer.go`. The new display name will propagate to any **newly created** folders. Already-existing folders retain their old names and would require a separate migration.

[Erik Andersson]: > "This is of course also relevant for any potential translation cases."

This PR was already approved by Erik prior to the session due to its small scope.

---

## Existing Incorrectly-Named Folders — Migration Discussion

Folders already created with the old integration ID names are not automatically fixed by this PR. A separate migration is needed.

### Current Situation for E Deal

- Estimated **2–3 E Deal customers** currently exist with incorrectly-named folders.
- No migration has been formally planned yet; Erik has not yet discussed it with **Emmanuel** (product owner for E Deal).
- Given the small customer count, a manual approach via API calls to Audience was initially considered.

### Migration Approach Debate

**[Erik Andersson]:** Suggested using direct API calls to Audience with access granted to the customer accounts. For CRM integration customers, there is generally a standing arrangement — the team just needs to ask SOC to grant account access, then use a real OAuth token for that account.

**[Michal Rosikiewicz]:** Pushed back on this approach. Requesting clients to grant account access just to rename a folder is a poor experience. Advocated for using an **internal delegated key from Apsis One (AO)** — an internal IP (integration platform) mechanism — which avoids needing to ask clients for access at all. This approach was used previously (via a "system key") but the system key method is no longer supported. The delegated internal key approach from AO is the current correct method.

**[Erik Andersson]:** Acknowledged he has no strong objection if the internal key approach is approved and available.

> ⚠️ **Ambiguity:** The exact mechanism for obtaining and using the internal delegated key from AO for folder rename migrations was not fully specified in this session. The speakers agreed in principle but deferred the details.

### Resolution

Michal will create a story to plan and execute the folder name migration using the internal delegated key approach.

---

## Leads — Upcoming Knowledge Transfer

Both speakers noted that lead handling deserves its own dedicated KT session. Topics deferred to that session include:
- How leads are gathered
- How lead handling interacts with outbound mappings
- Lead-to-contact merging: when a lead is no longer a lead and becomes a full contact, how does Apsis track the relationship between the lead profile and the contact profile?

---

## Key Takeaways

1. **Always use display name, not integration ID, for customer-visible folder names.** Integration IDs are internal contract identifiers and must not be exposed to customers.
2. **Integration IDs are immutable in practice.** Changing them requires coordinated API contract changes with the CRM vendor plus a full data migration. Display names are cheap to change.
3. **Connector configuration lives in `installer.go` (or `connector.go` for oldest connectors)** under `lib/connectors/<name>/`. This file contains both the integration ID and display name.
4. **Two folder creation paths exist:** lead entity folders (on install, for integrations with `additional entities` configured) and the legacy Maxo subscription sync folder (no longer active).
5. **To find which integrations support leads**, look for an `additional entities` block in `installer.go`. FSC Enterprise 12.0 does NOT have leads; FSC Corporate/E Deal, Dynamics, and Tribe do.
6. **Existing folders are not retroactively renamed** by this PR — a migration story needs to be created and executed for E Deal's 2–3 customers.
7. **Do not ask clients for account access just to run internal maintenance migrations.** Use the internal delegated key mechanism from AO instead.

---

## Unresolved Questions / Action Items

- [ ] **Michal:** Create a story for migrating existing incorrectly-named E Deal folders to use the display name `E Deal`.
- [ ] **Team:** Clarify and document the exact process for obtaining and using an internal delegated key from AO for Audience API calls in migration scripts.
- [ ] **Erik / Emmanuel:** Discuss with Emmanuel (E Deal product owner) whether the folder migration should be prioritized before Erik's departure.
- [ ] **Future KT session:** Deep dive on lead handling — gathering, outbound mappings, and lead-to-contact profile merging in Apsis.
- [ ] **Sprint/backlog planning:** Erik, Prem, and Lukasz to align on priority stories before Erik and Prem's departure; Michal's team may join planning or pull stories directly.