---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: ['Keyspace Architecture', 'CRM Entity Mapping', 'Lead vs Person Entities', 'Base Installer Pattern', 'Connector Inheritance', 'Keyspace Naming and Discrimination']
speakers: ['Erik Andersson', 'Lukasz Grabowski']
key_components: ['Base Installer', 'Generic Connector', 'Keyspace Function', 'E-deal (Efficy Corporate)', 'Microsoft Dynamics', 'Person Entity', 'Silhouette Entity', 'Discriminator Hashing']
session_type: knowledge-transfer
subdomains: ['Architecture', 'Different Types of Connectors', 'Generic Connector', 'E-deal (Efficy Corporate)', 'Lead creation', 'Duplicate profiles in Apsis']
---

## Session Overview

This session covers the foundational architecture of keyspaces in Apsis One integrations, focusing on how different CRM systems are modeled within the platform. Erik Andersson explains the distinction between person and lead entities (using E-deal's "silhouette" concept as an example), the role of the **Base Installer** class in providing shared functionality across all connectors, and the technical mechanism for generating unique keyspace identifiers through section discriminator hashing.

---

## CRM Entity Types: Person vs. Lead (Silhouette)

### Person Entity as Default Download Target

When downloading data from a CRM system like E-deal (Efficy Corporate), the default behavior is to download **person entities**. [Erik Andersson]:

> "If we say like download everything from E deal then we what we technically mean is we don't we will download all of the persons from the CRM system."

This is the primary entity type for established customers or contacts in the CRM.

### Lead Entity: Silhouette Concept in E-deal

E-deal uses the term **silhouette** to represent leads—incomplete profile data from form submissions. [Erik Andersson]:

> "CRM systems have some concept of a lead entity. In this case, EDIL has their silhouettes, so we typically don't download the silhouettes from the CRM or they don't send us profile updates for the lead up that is apart from deletion."

#### Why Silhouettes Matter

Silhouettes are created when a prospective customer submits a form on an Apsis-managed property. The flow works as follows:

1. Form submission event occurs
2. Apsis sends form data to the CRM
3. CRM responds: "We created something here" but indicates insufficient data to confirm if this is a full person or just a lead
4. CRM returns a **silhouette ID** (not a person ID)
5. Sales team can then reach out using this lead data

[Erik Andersson]:

> "Someone just essentially performed like they submitted a form saying like I'm interested but... you may not have like all information. It's possibly just like someone say I am interested in your products and like here is an e-mail address. You don't know anything else, but this is still enough because that means that your sales team can now set up a meeting with this person."

#### Critical Distinction in Apsis

It is critical that Apsis differentiates between a person entity and a silhouette entity in all downstream processing.

### Lead Generation and Automation

The typical workflow for lead generation involves:

1. Customer submits a form through Apsis
2. Lead (silhouette) is created in the CRM
3. Marketing automation (MA) flow triggers in the CRM to set up automated tasks
4. Example: Sales contact task scheduled within 7 days

[Erik Andersson]:

> "Someone submits a form, there's a lead created and now in the CRM there's automatically set up like someone in sales should contact this person like within seven days. Like all of that can be automated."

---

## Base Installer Pattern and Connector Inheritance

### Shared Logic via Base Installer Class

Rather than implementing identical logic across every connector (Dynamics, E-deal, Sideshop, Enterprise, FCC, Episerver), Apsis uses a **Base Installer** class to centralize common functionality.

[Erik Andersson]:

> "All of the generic connectors, they like inherit this generic dot installer function. There is actually one step even beneath that... all connectors will inherit from this base installer. So this is essentially like a blank boilerplate that fulfills all required functions."

#### When to Add Logic to Base Installer

If new functionality should apply uniformly across all CRM systems (e.g., a ping request to check connectivity), it is added to the Base Installer rather than duplicated across each connector implementation:

> "If you introduce a new function that says I want to be able to send a ping request to the CRM system. Then instead of you implementing this ping function on Dynamics, on Sideshop, on corporate, on enterprise, on FECU, in Episerver, like it's typically very necessary to duplicate that logic. Instead you add it here to the base installer where there is something that should work the same for all of the CRMS."

### Connector Inheritance Hierarchy

- **Base Installer**: Provides foundational, universal functions for all connectors
- **Generic Connector**: Inherits from Base Installer; adds generic connector-specific patterns
- **Specific Connectors** (Dynamics, E-deal, etc.): Inherit from Generic Connector or Base Installer directly

---

## Keyspace Architecture and Generation

### Purpose of Keyspaces

Keyspaces are the core organizational unit for managing CRM data within Apsis. Each installed CRM system generates a keyspace for its main entities.

[Erik Andersson]:

> "Every CRM system when it is installed, we will generate a key space for their like main entities. So for Microsoft Dynamics we gather the contacts here for E deal we gather their persons and pretty much all of the CRMS work exactly the same way."

### Keyspace Function in Base Installer

The **keyspace function** is defined on the Base Installer class and is inherited by all connectors:

> "You have a function here called key space. So this key space function is defined on the base installer. Every connector then inherits from this base installer. That means that every connector has access to this key space function and what it does is takes the configuration for the connector as input."

### Keyspace Configuration Parameters

The keyspace function takes the following configuration inputs:
- Connector configuration
- Account section
- Integration details

### Keyspace Naming Convention

The keyspace name is derived from the CRM system type. [Erik Andersson]:

> "We generate the name for it here and the name is the name of the CRM system. So if we install FSC Enterprise we will now create a the enterprise key space. If you create the Salesforce, we will create a Salesforce key space."

Examples:
- E-deal installation → "E deal" keyspace
- Microsoft Dynamics installation → "Dynamics" keyspace (implied from context)

---

## Keyspace Discriminator Generation

### Discriminator Purpose

The **discriminator** is the unique identifier for a keyspace within Apsis. It serves to distinguish one installation from another.

### Discriminator Format

All keyspace discriminators follow a consistent pattern:

```
integrations_keyspaces_<8-char-hash><logical_crm_name>
```

[Erik Andersson]:

> "All of our all of our key space discriminator are prepended here with integration key spaces. We then take the hash of the section discriminator like we hash it. I don't know if it like SHA one or something and then take the 1st 8 characters form it."

### Hash Generation Process

1. Take the section discriminator from the connector configuration
2. Hash it (likely SHA-1 or similar algorithm; exact algorithm not confirmed in session)
3. Extract the first 8 characters of the hash
4. Prepend the string `integrations_keyspaces_`
5. Append the logical name of the CRM system

### Example Discriminators

From the session, an example E-deal keyspace discriminator was shown:

```
integrations_keyspaces_<8-char-random-hash>_edeal
```

[Erik Andersson]:

> "So the discriminator here is like call matches 1 integrations, key spaces, lots of random characters, but these characters are the hash of this. The 1st 8 characters of the hash of the section discriminator and then we add the logical name of the of the CRM."

---

## Key Takeaways

1. **Entity Differentiation is Critical**: Apsis must clearly distinguish between person entities (established contacts) and lead entities (silhouettes from form submissions) because they have different data completeness and update patterns.

2. **Base Installer Drives Standardization**: The Base Installer class pattern eliminates code duplication across connectors by centralizing shared logic. New cross-connector functionality should be added here, not replicated in each connector.

3. **Keyspace as Organizational Unit**: Each CRM installation creates a unique keyspace named after the CRM system type. This keyspace organizes all main entity data for that installation.

4. **Deterministic Discriminator Generation**: Keyspace discriminators are generated deterministically by hashing the section discriminator and appending the CRM logical name. This ensures uniqueness and consistency across installations.

5. **Silhouettes Enable Early Sales Engagement**: E-deal's silhouette concept allows sales teams to engage with prospects immediately after form submission, before complete profile data is available, with automated CRM workflows managing follow-up tasks.

---

## Unresolved Questions

- [Erik Andersson] was uncertain about the exact hashing algorithm used for discriminator generation ("I don't know if it like SHA one or something").
- The exact structure and inheritance chain for connectors beyond "Base Installer → Generic Connector → Specific Connectors" was not fully detailed.