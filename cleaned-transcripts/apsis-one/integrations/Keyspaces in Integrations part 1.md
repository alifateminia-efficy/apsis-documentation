---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [Keyspace generation and naming, Entity types (Persons vs Leads/Silhouettes), CRM connector architecture, Base installer pattern, Lead entity handling, E-deal/FCC Corporate connector configuration]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Base Installer, Generic Connector, Keyspace function, CRM connectors (Microsoft Dynamics, E-deal/FCC Corporate, Salesforce), Account section, Integration configuration]
session_type: knowledge-transfer
subdomains: ['E-deal (Efficy Corporate)', 'Lead creation']
---

## Session Overview

This session covers the foundational concepts of **keyspace generation** in the Apsis One Integrations domain, with a focus on how different CRM systems are configured and how entities are differentiated. Erik Andersson explains the distinction between main entities (Persons) and lead entities (Silhouettes in E-deal), the inheritance-based architecture of CRM connectors, and the technical process by which keyspace discriminators are generated from account configuration. The discussion establishes why Apsis needs a structured approach to handle diverse CRM systems through a shared base installer pattern.

---

## Entity Types in CRM Systems

### Main Entities vs. Lead Entities

When a CRM system is integrated with Apsis, it typically has two primary entity types:

**Main Entities** (Persons/Contacts): These are fully-qualified customer records. When you say "download everything from E-deal," you are technically downloading all **Persons** from the CRM system. Microsoft Dynamics calls these **Contacts**; E-deal calls them **Persons**. These represent actual customers or prospects with sufficient data.

**Lead Entities** (Silhouettes in E-deal): CRM systems commonly have a concept of a lead entity—something less complete than a main entity. In E-deal, these are called **Silhouettes**. 

> A silhouette is a lead, something which someone has essentially performed—they submitted a form saying "I'm interested" but we don't know yet if this is a full customer or not.

[Erik Andersson]: The key distinction is that typically CRM systems do not download silhouettes, nor do they send profile updates for lead entities apart from deletion. Instead, silhouettes are gathered when a form submit event occurs. When a user submits a form, Apsis sends this event to the CRM, and the CRM responds: "We created something here, but we don't have enough data yet to say if this is an actual person or not. We have created a lead so your sales team can reach out to them."

### Form Submission and Lead Workflow

Lead creation flows from Apsis form submissions:

1. A user submits a form on an Apsis form (minimum required data: e-mail address)
2. Apsis sends this form submit event to the CRM system
3. The CRM creates a new lead entity (silhouette in E-deal)
4. The CRM responds with a **silhouette ID** (not a person ID)
5. Apsis must differentiate in its internal representation whether an entity is a Person or a Silhouette

[Erik Andersson]: "It is important in Apsis that we differentiate between them—if this is a person or if this is a silhouette."

Once the lead is created in the CRM, marketing automation workflows can be triggered:

> Someone submits a form, a lead is created, and in the CRM there's automatically set up like someone in sales should contact this person within seven days. All of that can be automated.

This allows the sales team to reach out proactively with minimal information, setting up meetings to present the product offering.

---

## Keyspace Architecture and Generation

### The Base Installer Pattern

The Apsis connector architecture uses an **inheritance-based pattern** to avoid duplicating logic across different CRM systems:

- **All connectors** inherit from the **Base Installer**
- The **Base Installer** is a blank boilerplate that fulfills all required functions
- All **Generic Connectors** inherit from the **Base Installer**

[Erik Andersson]: "Not all connectors are generic connectors, but all connectors will inherit from this base installer. So this is essentially a blank boilerplate that fulfills all required functions."

The reasoning is efficiency through centralization: If you introduce a new function (e.g., "send a ping request to the CRM system"), instead of implementing this ping function separately on Dynamics, Sideshop, Corporate, Enterprise, FECU, and Episerver—duplicating logic across all systems—you add it once to the Base Installer. This function will then work uniformly across all CRM connectors.

### The Keyspace Function

On the **Base Installer**, there is a function called **`keyspace`** that every connector inherits and thus has access to. 

[Erik Andersson]: "The keyspace function is defined on the base installer. Every connector then inherits from this base installer. That means that every connector has access to this keyspace function and what it does is takes the configuration for the connector as input—what is the account section and integration."

### Keyspace Naming and Discriminator Generation

When a CRM is installed, a keyspace is automatically generated for its main entities. The keyspace name is based on the CRM system type:

- Installing **FSC Enterprise** creates the **Enterprise keyspace**
- Installing **Salesforce** creates the **Salesforce keyspace**
- Installing **E-deal** creates the **E-deal keyspace**

The keyspace definition includes:

- **Name**: The name of the CRM system
- **Description**: A simple description
- **Discriminator**: The critical identifier

The **discriminator generation process** is standardized:

1. All keyspace discriminators are prepended with `integration.keyspaces.`
2. The **section discriminator** from the account configuration is hashed (using SHA-1 or similar)
3. The **first 8 characters** of the hash are taken
4. The logical name of the CRM is appended

**Example discriminator format**: `integration.keyspaces.[HASH_8_CHARS].[CRM_LOGICAL_NAME]`

[Erik Andersson]: "So the discriminator here is like call matches 1 integrations, keyspaces, lots of random characters, but these characters are the hash of the section discriminator and then we add the logical name of the CRM."

This approach ensures that each CRM installation gets a unique, reproducible keyspace discriminator based on its account configuration while remaining human-readable through the appended CRM name.

---

## Key Takeaways

1. **Entity Differentiation is Critical**: Apsis must distinguish between main entities (Persons) and lead entities (Silhouettes) because CRM systems handle them differently—leads don't receive profile updates and are created only via form submissions.

2. **Lead-to-Customer Journey**: Leads (silhouettes) are the entry point for form submissions; they contain minimal data but are sufficient for sales outreach. Full person records come later as leads are qualified.

3. **Inheritance Over Duplication**: The Base Installer pattern prevents code duplication across diverse CRM connectors (Dynamics, E-deal, Salesforce, etc.). New functionality is added once to the Base Installer and inherited by all connectors.

4. **Deterministic Keyspace Naming**: Keyspace discriminators are generated deterministically from account configuration (hashed section discriminator + CRM logical name), ensuring consistency and uniqueness across installations.

5. **Keyspaces Enable Multi-Tenancy**: By generating unique keyspace discriminators per CRM installation, Apsis can handle multiple CRM systems within a single instance without collision.

---

## Unresolved Questions / Notes for Follow-up

- The specific hash algorithm (SHA-1 or other) used for section discriminator hashing was mentioned uncertainly ("I don't know if it like SHA one or something") — confirm the actual algorithm used.
- Detailed explanation of how profile updates for person entities flow back from the CRM to Apsis was not covered in this session.
- The structure and purpose of the "account section" configuration was referenced but not fully elaborated.