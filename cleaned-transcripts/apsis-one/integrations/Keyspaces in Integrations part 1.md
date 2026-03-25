---
source_file: Keyspaces in Integrations part 1.txt
domain: Apsis One Integrations
topics: [Keyspace Generation, Entity Configuration, Lead vs Person Entity, CRM Connector Architecture, Base Installer Pattern]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace, Base Installer, Generic Connector, E-deal/Efficy Corporate, Microsoft Dynamics, Lead Entity, Person Entity, Section Discriminator, Hash Function]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, E-deal (Efficy Corporate), Lead creation]
---

## Session Overview

This is a technical deep-dive into how keyspaces work in the Apsis One Integrations platform, specifically focusing on entity configuration and keyspace generation mechanisms. Erik Andersson explains how different CRM systems (E-deal/Efficy Corporate, Microsoft Dynamics, etc.) are configured with entities, why lead vs. person entity distinction matters, and how the base installer pattern enables consistent keyspace generation across all connector types. The session covers the discriminator hashing algorithm and naming conventions used in production keyspace configuration.

## Entity Configuration in CRM Connectors

### Default Entities: Person vs. Lead

When installing a CRM connector (such as E-deal or Efficy Corporate), the system configures multiple entities. The default entity represents the main customer profile type:

- **Microsoft Dynamics**: Downloads **contacts**
- **E-deal/Efficy Corporate**: Downloads **persons**
- Lead entities are typically handled separately and are not downloaded in the same manner as person records

[Erik Andersson]: "If we say like download everything from E deal then we what we technically mean is we don't we will download all of the persons from the CRM system."

### The Silhouette Concept (E-deal Specific)

E-deal uses the concept of **silhouettes** as their lead entity type. A silhouette represents a prospect with incomplete information—typically someone who has submitted a form expressing interest but hasn't yet been qualified as a full customer.

[Erik Andersson]: "Silhouette is like a lead, some something which someone just essentially performed like they submitted a form saying like I'm interested..."

**Key distinction in Apsis**: It is critical to differentiate between whether a received record is a person or a silhouette, as they are handled differently in the platform.

### Lead Generation Workflow

The typical lead-to-customer flow works as follows:

1. A prospect submits a form through Apsis
2. This form submission event is sent to the CRM system
3. The CRM creates a **lead/silhouette** record (not a full person record)
4. The sales team receives a lead with minimal information (e.g., email address, expression of interest)
5. Sales can reach out and, upon conversion, the lead becomes a full person record
6. Profile updates from leads are not typically sent from the CRM unless the record is deleted

[Erik Andersson]: "They gather these silhouettes when you when we submit the form. So there's a form submit event. We send this to the CRM. They say, yeah, we created something here and like we don't have enough data yet to say if this is a actual person or not, but we have created a lead so. So our sales team can reach out to them..."

### Marketing Automation Integration with Lead Generation

Once a lead is created in the CRM, automated workflows can be triggered in the CRM system itself. For example:

- Automatic task creation: "Sales representative should contact this lead within 7 days"
- These workflows enable automated follow-up without manual intervention

[Erik Andersson]: "In the CRM there's automatically set up like someone in sales should contact this person like within seven days. Like all of that can be automated..."

---

## Keyspace Architecture and Generation

### Base Installer Pattern

The connector architecture uses an inheritance-based pattern to avoid code duplication across different CRM types:

- **All connectors** inherit from the **base installer**
- **Generic connectors** are a subset that also inherit from a **generic dot installer** function
- The base installer is a boilerplate that fulfills all required functions and provides a common foundation for CRM-agnostic logic

[Erik Andersson]: "Not all connectors are generic connector, but all connectors will inherit from this base installer. So this is essentially a blank boilerplate that fulfills all required functions."

### The Benefit of Centralizing Logic

Instead of implementing the same function across all connector types (Dynamics, Sideshop, Efficy Corporate, Enterprise, FECU, Episerver), new functions are added to the base installer once. This ensures consistency and reduces maintenance burden.

[Erik Andersson]: "If you introduce a new function that says I want to be able to... send a ping request to the CRM system... Instead of you implementing this ping function on Dynamics, on Sideshop, on corporate, on enterprise, on FECU, in Episerver, like it's typically very necessary to duplicate that logic. Instead you add it here to the base installer where there is something that should work the same for all of the CRMS."

### The Keyspace Function

The **keyspace function** is defined on the base installer and inherited by all connectors. It takes the connector configuration as input (account section and integration details) and generates the keyspace definition.

### Keyspace Naming Convention

The keyspace name is derived from the **CRM system name**:

- Efficy Enterprise installation → creates **"enterprise" keyspace**
- Salesforce installation → creates **"Salesforce" keyspace**
- E-deal installation → creates **"E-deal" keyspace** (based on the connector's logical name)

[Erik Andersson]: "The name is the name of the CRM system. So if we install FSC Enterprise we will now create the enterprise key space. If you create the Salesforce, we will create a Salesforce key space. The name is based on like what is the name for the connector here?"

### Keyspace Discriminator Generation Algorithm

The **discriminator** is the unique identifier for each keyspace and follows this algorithm:

1. **Prefix**: All keyspace discriminators are prepended with `integration.keyspaces`
2. **Hash Generation**: Take the section discriminator and hash it (algorithm appears to be SHA-1 or similar)
3. **Truncation**: Use only the first 8 characters of the hash
4. **Logical Name Suffix**: Append the logical name of the CRM system

**Format**: `integration.keyspaces.<8-char-hash>.<logical-crm-name>`

[Erik Andersson]: "All of our key space discriminator are prepended here with integration.keyspaces. We then take the hash of the section discriminator like we hash it. I don't know if it like SHA one or something and then take the 1st 8 characters from it... The discriminator here is like call matches 1 integrations, keyspaces, lots of random characters, but these characters are the hash of this. The 1st 8 characters of the hash of the section discriminator and then we add the logical name of the of the CRM."

### Example from Production

In production keyspace configuration, you can observe this pattern. For example, an E-deal keyspace discriminator would follow the pattern `integration.keyspaces.<truncated_hash>` with the CRM logical name appended.

---

## Key Takeaways

1. **Entity differentiation is critical**: Apsis must properly distinguish between person and lead entity types (silhouettes in E-deal's case) because they follow different update patterns and business rules.

2. **Lead generation workflow**: Leads enter the system via form submissions, exist with minimal data, and must be updated by the sales team before becoming full person records.

3. **Base installer inheritance enables consistency**: Rather than duplicating logic across connectors (Dynamics, Efficy, Salesforce, etc.), new CRM-agnostic functions are added to the base installer once and inherited by all connectors.

4. **Keyspace naming is deterministic**: Keyspace discriminators are algorithmically generated using section discriminator hashing (first 8 chars of hash) combined with the CRM logical name, ensuring consistency and uniqueness.

5. **Discriminator format**: `integration.keyspaces.<8-char-hash>.<crmname>` — this can be reverse-engineered to understand which configuration created a given keyspace.

---

## Unresolved Questions

- Hash algorithm specification: Erik was uncertain whether the hashing uses SHA-1 or another algorithm — this should be verified in code documentation
- Specific hash implementation location: The exact codebase location of the hash generation function was not pinpointed in the conversation