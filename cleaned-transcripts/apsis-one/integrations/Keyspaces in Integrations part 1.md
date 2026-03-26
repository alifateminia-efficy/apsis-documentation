---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [Keyspace generation, Entity types in CRM connectors, Lead vs Person entities, Connector architecture, Base installer pattern]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Base installer, Generic connectors, Keyspaces, E-deal connector, Microsoft Dynamics, Salesforce, Section discriminators]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Different Types of Connectors"]
---

## Session Overview

This session covers how Apsis One handles keyspace generation and entity management across different CRM integrations. Erik explains the distinction between **Person** and **Lead** (silhouette) entities in CRM systems, how the connector architecture uses a base installer pattern to avoid code duplication, and the technical process for generating unique keyspace discriminators based on configuration hashes. The discussion establishes foundational concepts needed to understand how Apsis One normalizes data across heterogeneous CRM platforms.

---

## Entity Types in CRM Connectors

### Person vs. Lead Entities

Different CRM systems model contacts differently. When integrating with a CRM:

- **Person**: The primary entity representing a known customer or contact. When downloading all data from a CRM system (e.g., "download everything from E-deal"), this technically means downloading all persons from that system.
- **Lead**: A preliminary contact entity created when someone submits a form or expresses initial interest, but before enough information is gathered to confirm they are a full customer.

[Erik Andersson]: In E-deal, leads are called **silhouettes**. CRM systems typically do not send profile updates for leads—they only send them during form submission events or deletion. They are designed as lightweight, incomplete contact records that allow sales teams to reach out and initiate engagement.

### Lead Generation Workflow

[Erik Andersson]: The lead generation flow works like this:

1. A customer submits a form via Apsis (forms capability)
2. The form submit event is sent to the CRM system
3. The CRM responds: "We've created a lead (silhouette), but we don't have enough data yet to confirm if this is an actual person"
4. The CRM sends back a **silhouette ID** (not a person ID)
5. Sales can now reach out to the lead with personalized messaging
6. Marketing automation (MA) flows can be triggered automatically—for example, setting a task for sales to contact this person within seven days

> It is important in Apsis that we differentiate between whether something is a person or whether it is a silhouette.

This distinction is critical in the Apsis system because the same contact may be represented as a lead in one state and upgraded to a full person later.

---

## Connector Architecture and Code Reuse

### The Base Installer Pattern

All connectors in Apsis One inherit from a **base installer**—a foundational class that provides common functionality shared across all CRM types (Dynamics, E-deal, Salesforce, Sideshop, Episerver, FSC Corporate, FSC Enterprise, etc.).

[Erik Andersson]: When you need to add a new capability that should work the same way across all CRM systems, you implement it once in the base installer instead of duplicating the logic across every connector. For example:

> If you introduce a new function that says "I want to be able to send a ping request to the CRM system," instead of implementing this ping function on Dynamics, Sideshop, Corporate, Enterprise, FECU, and Episerver, you add it here to the base installer where there is something that should work the same for all of the CRMs.

**Generic connectors** inherit from the base installer and extend it with standardized patterns that CRM vendors can implement.

### Keyspace Function

The **keyspace function** is defined on the base installer. Every connector inherits it, meaning every connector has access to keyspace generation logic. This function:

- Takes the connector configuration as input (account section, integration settings)
- Generates a unique identifier and name for the keyspace
- Uses the CRM system name as the basis for the keyspace name (e.g., "enterprise" for FSC Enterprise, "Salesforce" for Salesforce)

---

## Keyspace Naming and Discriminator Generation

### Discriminator Structure

Every Apsis keyspace has a **discriminator** that uniquely identifies it. The generation process:

1. All keyspace discriminators are prepended with: `integration.keyspaces`
2. The **section discriminator** from the connector configuration is hashed (SHA-1 or similar)
3. The **first 8 characters** of the hash are extracted
4. The **logical name** of the CRM system is appended

### Example

[Erik Andersson]: Looking at a live keyspace for E-deal:

```
integration.keyspaces.<8-char-hash>.edeal
```

Where `<8-char-hash>` is the first 8 characters of the hashed section discriminator, and `edeal` is the logical name of the connector.

This ensures that each installed CRM system gets a unique keyspace while maintaining a predictable, deterministic naming scheme.

---

## Key Takeaways

1. **Entity differentiation is fundamental**: Apsis must track whether a contact is a Person (full customer) or a Lead/Silhouette (early-stage prospect) because CRMs send different data and accept different operations for each.

2. **Code reuse via base installer**: The base installer pattern eliminates the need to duplicate logic across multiple connector implementations—any function that should work the same for all CRMs is implemented once at the base level.

3. **Deterministic keyspace naming**: Keyspace discriminators use hashed configuration values (section discriminator) combined with the CRM system name. This ensures uniqueness and repeatability—the same connector configuration always generates the same keyspace.

4. **Lead generation is a core workflow**: Form submission → Lead creation in CRM → Sales outreach → MA automation is a standard pattern that Apsis orchestrates across different CRM systems.

---

## Unresolved Questions / Clarifications

- The exact hashing algorithm (SHA-1 suspected but not confirmed) used for section discriminator → 8-character prefix
- Whether the keyspace name suffix uses the "logical name" of the connector or a different identifier in all cases