---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [Keyspace architecture, Lead entity handling, CRM entity mapping, Integration configuration, Base installer pattern]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Base Installer, Generic Connector, Keyspace function, E-deal connector, Microsoft Dynamics connector, CRM system configuration]
session_type: knowledge-transfer
subdomains: ['Architecture', 'Lead creation', 'E-deal Integration']
---

## Session Overview

Erik Andersson explains the foundational architecture of how Apsis One manages keyspaces across different CRM integrations. The session covers how multiple CRM systems (E-deal, Microsoft Dynamics, Episerver, etc.) are configured with separate keyspaces, how the **lead entity** concept differs between CRM systems, and the inheritance hierarchy that allows the **base installer** to define common functionality used by all connectors. The discussion focuses on how keyspace names and discriminators are generated algorithmically based on connector configuration.

---

## Lead Entity Concept and CRM Entity Types

### Entity Types in CRM Systems

[Erik Andersson]: When we say "download everything from E-deal," we technically mean downloading all of the **persons** from the CRM system. This is the main entity that Apsis defaults to.

However, CRM systems commonly have a concept of a **lead entity**, which is separate from the main person/contact entity. E-deal calls these leads **silhouettes**. 

### Silhouettes as Lead Representations

[Erik Andersson]: A **silhouette** is a lead — something created when someone submits a form expressing interest, but we don't yet have enough data to confirm if this is an actual customer. The key difference in Apsis is that we must differentiate between a person and a silhouette.

[Lukasz Grabowski asked for clarification on the meaning of "silhouette" — it's a term used by E-deal for lead entities.]

### Lead Generation and Profile Creation Flow

When a form is submitted through Apsis:
1. A form submit event is triggered and sent to the CRM system
2. The CRM creates a lead entity (E-deal returns a **silhouette ID**, not a person ID)
3. Unlike persons, CRM systems typically don't send profile updates for leads — only deletion notifications
4. The lead contains minimal information: possibly just an email address and expression of interest
5. This minimal data is sufficient for the sales team to initiate outreach

[Erik Andersson]: The sales team can set up meetings with these leads by saying, "I saw that you were interested in our product."

### Automated Marketing Workflows for Leads

Apsis customers typically use forms to gather interest and create leads in the CRM. A **marketing automation (MA) flow** can be configured to automatically set up sales tasks. For example: "If a lead is created, automatically create a task for someone in sales to contact this person within seven days."

---

## Keyspace Architecture and Configuration

### The Need for Keyspaces in Apsis

[Erik Andersson]: Every CRM system, when installed, is assigned a keyspace for its main entities. For Microsoft Dynamics, we gather the **contacts**; for E-deal, we gather **persons**. Pretty much all CRM systems work exactly the same way in terms of keyspace generation.

The reason keyspaces exist: Apsis handles everything with keyspaces as a fundamental data organization concept.

### Base Installer and Inheritance Hierarchy

The connector architecture follows an inheritance pattern:

1. **Base Installer** — A blank boilerplate that fulfills all required functions. Every connector in the system inherits from this.
2. **Generic Connector** — Not all connectors are generic, but all connectors inherit from the **base installer**.
3. **Specific Connectors** — Implementations for individual CRM systems (Dynamics, E-deal, Sideshop, Corporate, Enterprise, FECU, Episerver, Salesforce, etc.)

[Erik Andersson]: Instead of duplicating a new function across all CRM-specific connectors (Dynamics, Sideshop, Corporate, Enterprise, FECU, Episerver), we implement it once in the **base installer**. If we introduce a new function — for example, "send a ping request to the CRM system" — it's typically necessary to avoid duplicating that logic across every single connector.

### The Keyspace Function

The **base installer** contains a `keyspace` function that is inherited by every connector. This function:
- Takes the connector configuration as input (specifically: the account section and integration section)
- Generates the definition for how the audience should use the keyspace
- Creates a name and discriminator for the keyspace

### Keyspace Naming Convention

The keyspace name is based on the CRM system name:
- Installing "FSC Enterprise" creates an "Enterprise keyspace"
- Installing "Salesforce" creates a "Salesforce keyspace"
- Installing "E-deal" creates an "E-deal keyspace"

The name is derived from the connector's name property in the configuration.

### Keyspace Discriminator Generation

[Erik Andersson]: The discriminator is the important part of the keyspace identification. The generation process:

1. All keyspace discriminators are **prepended with `integration_keyspaces.`**
2. The section discriminator is then **hashed** (Erik mentions this might be SHA-1, but doesn't confirm the exact algorithm)
3. The first 8 characters of the hash are taken
4. These 8 characters are combined with the logical name of the CRM system

**Example from Apsis system**: The E-deal keyspace discriminator appears as something like `integration_keyspaces_[first_8_hash_chars]_e-deal`, where the hash characters are derived from the section discriminator hash.

[Erik Andersson]: When you look at keyspaces in the system, you see many keyspaces listed. For example, the E-deal one has a discriminator like `integration_keyspaces_` followed by what looks like random characters. These characters are actually the first 8 characters of the hash of the section discriminator, followed by the logical name of the CRM.

---

## Key Takeaways

1. **Lead entities are critical for early-stage prospect management** — Silhouettes/leads in E-deal are distinct from persons and represent prospects with minimal data. Apsis must differentiate between them.

2. **Keyspaces provide isolation per CRM installation** — Each CRM system gets its own keyspace, named after the system and identified by a hashed discriminator.

3. **Base Installer provides reusable functionality** — Common connector logic is implemented once in the base installer and inherited by all CRM-specific connectors, avoiding duplication.

4. **Keyspace discriminators use deterministic hashing** — The discriminator is generated by hashing the section discriminator, taking the first 8 characters, and appending the CRM logical name. This ensures consistent identifiers.

5. **Marketing automation can be triggered on lead creation** — Apsis enables workflows where form submissions create leads in the CRM, which then automatically trigger sales activities.

---

## Unresolved Questions / Clarifications Needed

- **Exact hashing algorithm**: Erik mentions the hash might be SHA-1 but doesn't confirm definitively. Need to verify the exact algorithm used for section discriminator hashing.
- **Practical examples**: The discriminator generation logic would benefit from a concrete worked example showing the input section discriminator and resulting keyspace discriminator.