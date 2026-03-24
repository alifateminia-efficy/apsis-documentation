---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One - Integrations
topics: [Keyspace Architecture, Entity Mapping in CRM Integrations, Lead vs Person Entity Distinction, Base Installer Pattern, Keyspace Naming and Discriminator Generation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Base Installer, Generic Connector, Keyspace Function, CRM Connectors (E-deal, Microsoft Dynamics, Salesforce, FSC Enterprise), Lead Entity, Person Entity]
session_type: knowledge-transfer
---

## Session Overview

This session covers the foundational architecture of keyspaces in the Apsis One Integrations domain. Erik Andersson explains how different CRM systems are integrated into Apsis, focusing on how entities (particularly Person and Lead entities) are mapped and stored in keyspaces. The discussion covers the inheritance pattern used across connectors, the standard keyspace generation logic, and the critical distinction between lead entities (silhouettes in E-deal terminology) and full person records.

---

## Entity Types and Lead vs Person Distinction

### Person and Lead Entities in CRM Systems

When integrating with a CRM system like E-deal, there are typically two main entity types to configure:

1. **Person Entity** — The primary entity type that represents a known customer or contact with substantial data. When you say "download everything from E-deal," what you technically mean is downloading all **Persons** from the CRM system.

2. **Lead Entity** — A preliminary entity representing a prospect or interested party. E-deal calls their lead entity a **"silhouette"**. These are created when a form submission occurs and represent someone interested in the product but for whom insufficient data exists yet to confirm they are an actual customer.

### The Silhouette Concept in E-deal

[Erik Andersson]: A silhouette is essentially a lead — it represents someone who has submitted a form indicating interest but hasn't yet been fully qualified as a customer. The key workflow is:

1. A user submits a form on the Apsis platform
2. This form submission event is sent to the CRM system (E-deal in this example)
3. E-deal responds: "We've created a lead/silhouette, but we don't have enough data yet to say if this is an actual person"
4. The silhouette is assigned a **silhouette ID** (not a person ID), which is critical for Apsis to track
5. E-deal does not typically send profile updates for leads/silhouettes, except for deletion events
6. The sales team uses this silhouette data to reach out to the prospect

> It is important in Apsis that we differentiate between whether this is a Person or whether this is a Silhouette.

### Lead Generation and CRM Automation

The typical lead generation workflow involves:
- Apsis forms are used to capture interest and create leads in the CRM system
- Once a lead is created in the CRM, Marketing Automation (MA) flows can automatically trigger tasks
- Example: A task could be automatically created saying "Sales should contact this person within seven days"

---

## Keyspace Architecture and Inheritance Pattern

### Base Installer and Connector Inheritance

All CRM connectors in Apsis follow an inheritance hierarchy:

**Base Installer** (lowest level)
- Defines required functions that should work identically across all CRM systems
- Provides a blank boilerplate that fulfills all required functions
- Any function that should have consistent behavior across all CRM types is implemented here

**Generic Connector** (intermediate level)
- Inherits from Base Installer
- Not all connectors are generic connectors, but all connectors inherit from Base Installer

**Individual Connectors** (e.g., Microsoft Dynamics, E-deal, Salesforce, FSC Enterprise)
- Inherit from either Generic Connector or Base Installer
- Can override or extend behavior as needed for their specific CRM system

### Why This Inheritance Pattern Matters

[Erik Andersson]: The inheritance pattern prevents code duplication. If you need to add a new function that should work the same across all CRM systems (for example, a ping request function), you implement it once in the Base Installer instead of replicating it across:
- Microsoft Dynamics connector
- Sideshop connector
- Corporate connector
- Enterprise connector
- FSC connector
- Episerver connector

---

## Keyspace Generation and Naming

### The Keyspace Function

The **keyspace function** is defined on the Base Installer and therefore available to every connector that inherits from it. 

**Inputs:**
- Connector configuration
- Account section information
- Integration information

**Outputs:**
- Keyspace definition for storing that CRM's entities

### Keyspace Naming Convention

When a CRM connector is installed, a keyspace is automatically created. The name follows a standard pattern:

```
[CRM System Name] Keyspace
```

Examples:
- Installing **FSC Enterprise** creates: `Enterprise Keyspace`
- Installing **Salesforce** creates: `Salesforce Keyspace`
- Installing **E-deal** creates: `E-deal Keyspace`

The name is derived from the logical connector name from the configuration.

### Keyspace Discriminator Generation

The **discriminator** is the unique identifier for a keyspace in Apsis. The generation process is:

1. **Base prefix:** All keyspace discriminators are prepended with `integration.keyspaces.`
2. **Hashing:** The section discriminator (from the connector configuration) is hashed using a hashing algorithm (believed to be SHA-1 or similar)
3. **Truncation:** Take the first 8 characters of the hash
4. **Appending:** Append the logical name of the CRM system

**Example structure:**
```
integration.keyspaces.[first 8 chars of hash][CRM logical name]
```

For instance, the E-deal keyspace discriminator might look like:
```
integration.keyspaces.c0ll.ma7c.edeal
```
(where the alphanumeric characters after "integration.keyspaces" represent the 8-character hash and the CRM name)

---

## Key Takeaways

1. **Entity distinction is critical**: Apsis must differentiate between Person entities and Lead entities (called Silhouettes in E-deal) because they have different lifecycles and update patterns.

2. **Lead entities don't receive profile updates**: CRM systems typically do not send profile updates for leads/silhouettes except for deletion events — the data is static until the lead is converted to a person.

3. **Form submission is the entry point**: Lead creation in CRM systems is typically triggered by Apsis form submissions, enabling integration with CRM workflows and marketing automation.

4. **Base Installer ensures consistency**: By implementing common functionality in the Base Installer (not in individual CRM connectors), the system avoids code duplication and ensures consistent behavior across all integrated CRM systems.

5. **Keyspace naming is deterministic**: Keyspaces are automatically generated with names based on the CRM system name, and unique discriminators are created using hashed section identifiers, making them deterministic and traceable.

6. **Configuration-driven generation**: The entire keyspace setup is driven by the connector configuration — no manual setup is required when installing a new CRM connector.

---

## Unresolved Questions / Clarifications Needed

- Confirm the exact hashing algorithm used for keyspace discriminator generation (mentioned as "SHA one or something")
- Clarify the specific structure of the "section discriminator" and how it maps to the connector configuration
- Verify whether all CRM systems follow the same lead/person entity pattern, or if some have variations beyond what was discussed