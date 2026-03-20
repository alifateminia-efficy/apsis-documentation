---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One - Integrations
topics: [Keyspace Architecture, Entity Models, CRM Integration, Lead vs Contact Entity Distinction, Base Installer Pattern, Keyspace Naming and Discriminator Generation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Base Installer, Generic Connector, Keyspace, CRM Connectors (EDeal, Microsoft Dynamics, Salesforce, FSC Enterprise), Entity Models (Person, Contact, Silhouette/Lead)]
session_type: knowledge-transfer
---

## Session Overview

This session covers the foundational architecture of how Apsis One manages keyspaces for different CRM integrations. Erik Andersson walks through the entity models used across various CRM systems (EDeal, Microsoft Dynamics, Salesforce), the distinction between different entity types (persons vs. leads/silhouettes), and the technical mechanism for generating and naming keyspaces through the Base Installer pattern. The discussion establishes why keyspace differentiation matters and how the system's inheritance hierarchy ensures consistent functionality across all connector implementations.

---

## Entity Models and CRM Data Concepts

### Person vs. Lead/Silhouette Entity Distinction

[Erik Andersson]: When integrating with CRM systems like EDeal, there are typically two main entities configured. The primary entity is **Person**, which represents a fully qualified customer record. If you download everything from EDeal, what you technically mean is downloading all Persons from the CRM system.

However, most CRM systems have a concept of a **lead entity**. In EDeal's case, these are called **silhouettes**. These are handled differently from Persons:

- Silhouettes are typically not downloaded from the CRM
- Profile updates for silhouettes are not received (except for deletion events)
- They are created when a form is submitted

### Lead Generation Workflow

[Erik Andersson]: A silhouette is essentially a lead—someone who has shown initial interest but you don't yet know if they are a full customer. The typical scenario:

1. A user submits a form, indicating interest in the product
2. The CRM system creates a lead (silhouette in EDeal terminology) with minimal data—possibly just an email address
3. In Apsis, this triggers a marketing automation (MA) flow to set up automated tasks
4. For example, a sales task can be automatically created: "Contact this person within 7 days"

> The key point is that in Apsis, we must differentiate between whether an entity is a Person or a Silhouette, as this affects how we process and route the data.

**Why this matters**: Apsis handles everything through keyspaces, and every CRM system installation generates a keyspace for main entities. For Microsoft Dynamics, the main entity is Contact. For EDeal, it's Person. The lead entity (silhouette) handling is separate and requires explicit processing logic.

---

## Base Installer and Connector Architecture

### Inheritance Hierarchy

[Erik Andersson]: All CRM connectors follow a strict inheritance pattern to avoid code duplication across different CRM implementations (Microsoft Dynamics, EDeal, Salesforce, FSC Enterprise, Episerver, etc.).

The hierarchy is:
- **Base Installer** (top level) - provides required function boilerplate
- **Generic Connector** - inherits from Base Installer
- **Specific Connectors** (Dynamics, EDeal, Salesforce, etc.) - inherit from Generic Connector or directly from Base Installer

### Why This Pattern Matters

[Erik Andersson]: When a new function needs to be added that works consistently across all CRMs (for example, a ping request to the CRM system), instead of implementing it separately on Dynamics, EDeal, Salesforce, FSC Enterprise, Episerver, and every other connector, you implement it once in the Base Installer. Every connector then automatically inherits this functionality.

This is particularly important for the **keyspace function**, which is defined on the Base Installer and ensures consistent keyspace generation behavior across all CRM integrations.

---

## Keyspace Generation Mechanism

### The Keyspace Function

[Erik Andersson]: The `keyspace` function is defined on the Base Installer and takes the following inputs:
- The connector configuration
- The account section
- The integration configuration

The function generates:
1. **Name**: Based on the CRM system name (e.g., "Salesforce keyspace" if you install Salesforce; "Enterprise keyspace" if you install FSC Enterprise)
2. **Discriminator**: The unique identifier for the keyspace (this is the critical component for differentiation)

### Discriminator Generation Algorithm

[Erik Andersson]: All keyspace discriminators follow this pattern:

```
integrations_keyspaces_[8-character hash]_[logical CRM name]
```

The algorithm works as follows:

1. Start with the prefix: `integrations_keyspaces`
2. Hash the section discriminator (using SHA-1 or similar hashing algorithm)
3. Take the first 8 characters of the hash
4. Append the logical name of the CRM system

**Example from the session**: For an EDeal integration, the discriminator appears as:
```
[hash matches]_integrations_keyspaces_[random characters]_edeal
```

Where the random characters are the first 8 characters of the hashed section discriminator, followed by the CRM logical name.

### Why Hash the Discriminator?

[Erik Andersson]: The hash provides a stable, unique identifier based on the section configuration while keeping the discriminator length reasonable. This prevents collisions and makes the keyspace identifiable even when the same connector is installed multiple times with different configurations.

---

## Keyspace Naming Convention Across CRM Systems

[Erik Andersson]: Every CRM system, when installed, generates a keyspace for its main entity:

- **Microsoft Dynamics** → Contact keyspace
- **EDeal** → Person keyspace  
- **Salesforce** → Account/Contact keyspace (depending on configuration)
- **FSC Enterprise** → Enterprise keyspace

The name generation is consistent because all connectors inherit from the Base Installer's keyspace function. The CRM system name becomes part of the keyspace name automatically based on what connector is being installed.

---

## Key Takeaways

1. **Entity differentiation is critical in Apsis**: Persons and Silhouettes (leads) must be handled differently because they represent different stages of the customer journey and have different data completeness profiles.

2. **The Base Installer pattern ensures consistency**: Rather than reimplementing functions across multiple CRM connectors, shared logic lives in the Base Installer, making the codebase more maintainable.

3. **Keyspace generation is automated and deterministic**: The `keyspace` function generates consistent, hashed discriminators following a predictable pattern (`integrations_keyspaces_[hash]_[crmname]`), allowing the system to identify and route data to the correct keyspace regardless of CRM type.

4. **Keyspace names map to CRM system names**: When you install a new CRM connector, the keyspace name automatically reflects the CRM system name, making the mapping intuitive for system administrators.

5. **Silhouettes enable lead generation workflows**: By treating lead entities separately from contacts/persons, Apsis can support the full lead-to-customer lifecycle without loading incomplete records into the main contact/person keyspace.

---

## Unresolved Questions / Points for Clarification

- The specific hashing algorithm used (SHA-1 or another variant) for discriminator generation was mentioned as uncertain ("I don't know if it like SHA one or something")
- The exact structure of "section discriminator" and how it's populated could benefit from a detailed walk-through
- Whether the 8-character hash truncation has ever caused collisions in production was not discussed