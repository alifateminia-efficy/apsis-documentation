---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [keyspace generation, CRM connectors, lead entities, silhouette concept, base installer inheritance, discriminator naming convention]
speakers: ["Erik Andersson (explainer/senior developer)", "Lukasz Grabowski (learner/new developer)"]
key_components: [eDeal connector, FCC Corporate connector, Microsoft Dynamics connector, Salesforce connector, FSC Enterprise connector, base installer, generic connector, key space function]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson walks Lukasz Grabowski through how CRM integrations are structured in Apsis One, focusing on how key spaces are generated and named for CRM connector installations. The session covers the eDeal/FCC Corporate connector as a concrete example, explains the concept of lead entities (called "silhouettes" in eDeal), and walks through the inheritance chain from individual connectors up to the base installer where the shared `keyspace` function lives. The naming and discriminator generation strategy for key spaces is explained in detail.

---

## CRM Entity Types: Persons vs. Silhouettes in eDeal

### Main Entity: Person

When downloading everything from eDeal, the default entity type is **Person** — this is the primary profile concept in the CRM. If you say "download everything from eDeal," what that technically means is: download all Persons from the CRM system.

### Lead Entity: Silhouette

eDeal has a concept called a **silhouette**, which is their term for a lead — someone who has expressed interest but for whom there is not yet enough data to confirm they are a full customer/contact.

> "Silhouette is like a lead — something where someone just essentially submitted a form saying 'I'm interested,' but you don't know yet if this is a full customer or not."

**How silhouettes are created in practice:**
1. A customer submits a form in Apsis (customers use Apsis forms for lead generation).
2. Apsis sends a form-submit event to eDeal.
3. eDeal creates a silhouette and responds with a **silhouette ID** (not a person ID).
4. The silhouette ID is returned to Apsis.

**Key distinction:** Apsis must differentiate between a Person and a Silhouette. They have different IDs and different meanings downstream.

**What Apsis does NOT do with silhouettes:** Apsis does not download silhouettes from the CRM, and silhouettes do not trigger profile updates — except for deletion events.

### Why This Matters for Sales Automation

Once a silhouette is created in eDeal, the MA (Marketing Automation) flow can automate tasks — for example, automatically assigning a sales rep to contact the lead within seven days. This entire pipeline (form submit → silhouette creation → automated task assignment) is orchestrated through the integration.

---

## Connector Architecture: Base Installer Inheritance

### Inheritance Chain

Not all connectors are **generic connectors**, but all connectors inherit from the **base installer**. The hierarchy is:

```
Base Installer
  └── Generic Connector (inherits base installer)
        └── Specific Connectors (e.g., eDeal, Microsoft Dynamics, Salesforce, FSC Enterprise, Episerver, Sideshop)
```

### Purpose of the Base Installer

The base installer acts as a blank boilerplate that fulfills all required connector functions. If a new shared function needs to be added — for example, a "ping request to the CRM system" — it should be added to the base installer rather than duplicated across every individual connector (Dynamics, Sideshop, Corporate, Enterprise, FCC, Episerver, etc.).

> "Instead of implementing this ping function on Dynamics, on Sideshop, on Corporate, on Enterprise, on FCC, on Episerver — it's typically very necessary [not] to duplicate that logic. Instead you add it here to the base installer where there is something that should work the same for all of the CRMs."

---

## Key Space Generation for CRM Connectors

### When Key Spaces Are Created

Every time a CRM connector is **installed**, Apsis generates a key space for that connector's main entity type. Examples:

| Connector Installed | Key Space Created |
|---|---|
| Microsoft Dynamics | Contacts key space |
| eDeal | Persons key space |
| FSC Enterprise | Enterprise key space |
| Salesforce | Salesforce key space |

### The `keyspace` Function

The `keyspace` function is defined on the **base installer**. Because every connector inherits from the base installer, every connector has access to this function.

**Inputs to the function:**
- The connector configuration
- The account section
- The integration

### Key Space Name

The **name** of the key space is derived from the logical name of the CRM system (e.g., "eDeal" if you install the eDeal connector).

### Discriminator Generation

The **discriminator** is the more technically important field. It is constructed as follows:

```
integrations.keyspaces.<first 8 characters of hash of section discriminator><logical name of CRM>
```

- All integration key space discriminators are **prepended with** `integrations.keyspaces.`
- The unique portion is the **first 8 characters of a hash** (Erik believes SHA-1 or similar) of the **section discriminator**
- Followed by the **logical name** of the CRM connector

**Example (eDeal):** Looking at the eDeal key space in the system, the discriminator follows this pattern:

```
integrations.keyspaces.<8-char-hash>edeal
```

> "The discriminator here is like... `integrations.keyspaces`, lots of random characters, but these characters are the hash — the first 8 characters of the hash of the section discriminator — and then we add the logical name of the CRM."

⚠️ **Ambiguity:** Erik was not certain of the exact hash algorithm used ("SHA-1 or something"). This should be confirmed by checking the base installer source code directly.

---

## Key Takeaways

1. **Persons vs. Silhouettes:** eDeal uses "silhouette" for what Apsis conceptually calls a lead. Apsis handles silhouettes differently from persons — they are created via form-submit events and return a silhouette ID. Apsis does not pull silhouettes from eDeal proactively.
2. **Base installer is the shared foundation:** All connectors inherit from the base installer. Shared logic (including key space generation) lives there to avoid duplication across connector implementations.
3. **Key spaces are auto-generated on install:** Every CRM connector installation triggers creation of a key space for its main entity type via the `keyspace` function on the base installer.
4. **Discriminator format:** `integrations.keyspaces.` + first 8 chars of hash of section discriminator + logical CRM name. This format is consistent across all CRM integrations.
5. **Forms as lead intake:** Apsis customers use Apsis forms as the primary mechanism for lead generation, feeding into CRM lead/silhouette creation.

---

## Unresolved Questions

- **Hash algorithm:** The exact algorithm used to hash the section discriminator for key space discriminator generation was not confirmed. Erik said "SHA-1 or something" — verify in base installer source code.
- **Session continues:** This is Part 1 — the walkthrough of key spaces in integrations is not complete. Part 2 likely covers additional detail.