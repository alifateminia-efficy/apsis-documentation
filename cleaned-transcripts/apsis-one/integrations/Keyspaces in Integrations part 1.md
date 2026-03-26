---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [keyspace generation, lead creation, CRM entity types, base installer inheritance, discriminator hashing, connector architecture]
speakers: ["Erik Andersson", "Lukasz Grabowski"]
key_components: [Apsis One, E-deal/EDIL, Microsoft Dynamics, FSC Enterprise, Salesforce, Episerver, Sideshop, base installer, generic connector, key spaces]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Different Types of Connectors"]
---

## Session Overview

Erik Andersson walks Lukasz Grabowski through how Apsis One handles CRM integrations, focusing on two related concepts: (1) how leads (called "silhouettes" in E-deal) are created when a form is submitted and why they are treated differently from full person records, and (2) how key spaces are generated per connector using a base installer class and a hashed discriminator scheme. The session uses E-deal (EDIL) as the primary example but notes the pattern applies across all connectors. This appears to be part one of a multi-part session.

---

## CRM Entity Types: Persons vs. Leads (Silhouettes)

### Main Entity Configuration

In the E-deal/FSC Corporate connector, two entities are configured:

- **Person** — the main entity. "Downloading everything from E-deal" technically means downloading all `Person` records from the CRM.
- **Silhouette** — E-deal's term for a **lead**: a contact record created when someone submits a form, but where there is not yet enough data to confirm they are a full person/customer.

> "They gather these silhouettes when you submit the form. So there's a form submit event — we send this to the CRM. They say, 'we created something here and we don't have enough data yet to say if this is an actual person or not, but we have created a lead.'"

### Silhouette vs. Person ID — Why the Distinction Matters

When a form is submitted:
1. Apsis sends the form submission event to the CRM.
2. The CRM responds with a **silhouette ID** (not a person ID).
3. It is important in Apsis to differentiate between a `Person` record and a `Silhouette` record.

[Erik Andersson]: Silhouettes (leads) are **not downloaded** from the CRM as profile updates — the only exception is deletions. They are created via the form submit flow, not via the standard sync.

### Business Context for Leads

A silhouette/lead represents someone who has expressed interest (e.g., submitted a form with just an email address). The sales team can then reach out. The Marketing Automation (MA) flow can automate follow-up tasks:

> "Someone submits a form, there's a lead created, and now in the CRM there's automatically set up: 'someone in sales should contact this person within seven days.' All of that can be automated."

---

## Key Space Generation per Connector

### Why Key Spaces Exist

Every CRM connector, when installed, generates a **key space** for its main entities. This is how Apsis tracks identifiers from each CRM system:

- Microsoft Dynamics → key space for `Contact` records
- E-deal → key space for `Person` records
- The pattern is consistent across virtually all CRM connectors.

### Base Installer and Inheritance Hierarchy

[Erik Andersson]: There is a hierarchy of installer classes:

1. **`BaseInstaller`** — the root class. All connectors inherit from this. It is a blank boilerplate that fulfills all required functions.
2. **`GenericConnector` installer** — inherits from `BaseInstaller`. Generic connectors add another layer on top.
3. **Individual connector installers** (e.g., Dynamics, E-deal, Salesforce, FSC Enterprise, Episerver, Sideshop) — inherit from the appropriate level.

The rationale for `BaseInstaller`:

> "Instead of you implementing a function on Dynamics, on Sideshop, on Corporate, on Enterprise, on FECU, on Episerver — it's typically very necessary to not duplicate that logic. Instead you add it here to the BaseInstaller where there is something that should work the same for all of the CRMs."

### The `key_space` Function

A function called **`key_space`** is defined on `BaseInstaller`. Because every connector inherits from `BaseInstaller`, every connector has access to this function.

**Inputs:** The connector's configuration — specifically the account section and integration.

**Output:** A key space definition in the format Audience (Apsis) expects, including:
- A **name** based on the CRM system name (e.g., installing E-deal creates an "E-deal" key space; installing FSC Enterprise creates an "Enterprise" key space; installing Salesforce creates a "Salesforce" key space).
- A **description** (simple, less important).
- A **discriminator** (the critical field).

### Discriminator Format and Hashing Scheme

All key space discriminators follow this structure:

```
integrations.key_spaces.<first_8_chars_of_hash>.<logical_crm_name>
```

- Prepended with `integrations.key_spaces`
- Followed by the **first 8 characters of a hash** of the section discriminator (Erik believes it is SHA-1 or similar — ⚠️ exact algorithm not confirmed in this session)
- Followed by the **logical name** of the CRM

[Erik Andersson]: Looking at the E-deal key space in the system, the discriminator reads as something like `integrations.key_spaces.<random_8_chars>.edeal` — the "random" characters are deterministic: they are the first 8 characters of the hash of the section discriminator.

---

## Key Takeaways

- E-deal uses the term **silhouette** for what Apsis conceptually calls a **lead** — a partial contact record created on form submission, not yet a confirmed person.
- Silhouettes/leads are **created** in the CRM via form submit events but are **not synced back** to Apsis as profile updates (except for deletions).
- All connectors inherit from **`BaseInstaller`**, which defines shared logic including the `key_space` function, avoiding duplication across connector implementations.
- Key space **discriminators** are deterministically generated: `integrations.key_spaces` + first 8 chars of a hash of the section discriminator + logical CRM name.
- The key space name is human-readable and based on the CRM's name; the discriminator provides the stable unique identifier.

---

## Unresolved Questions

- ⚠️ The exact hashing algorithm used for the discriminator was not confirmed — Erik said "SHA-1 or something." This should be verified in the codebase.
- The session is labeled "part 1" — key space behavior for connectors that are **not** generic connectors (i.e., legacy and third-party connectors) was not fully covered and may be addressed in a follow-up session.