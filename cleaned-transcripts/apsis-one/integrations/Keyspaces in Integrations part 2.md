---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One - Integrations
topics: [Keyspace Architecture, CRM Discriminators, Entity Key Spaces, Lead vs Contact Handling, Form Submission Events, CRM-Specific Behavior, Profile Merging, Data Integrity]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace Discriminators, Installation Manager, Entity Key Spaces, Lead Key Spaces, Form Submission Events, Profile Merging, E-deal, FSC Corporate, FSC Enterprise 12.1, Tribe, Dynamics]
session_type: knowledge-transfer
---

## Session Overview

This session covers the second part of keyspace architecture in the Apsis One integrations domain. The discussion focuses on how keyspace discriminators are constructed and used across different CRM systems, the distinction between main contact keyspaces and entity-specific keyspaces (particularly for leads and silhouettes), how form submissions are processed and routed to different CRM entity types, and the critical importance of avoiding profile merges that cross CRM boundaries. The conversation highlights why certain design decisions were made and the data integrity risks associated with improper merging of profiles sourced from CRM systems.

---

## Keyspace Discriminator Structure and Uniqueness

### The Discriminator Format

The keyspace discriminator follows a consistent pattern across all CRM installations:

```
integrations:keyspaces:<8-char-hash>:<crm-logical-name>
```

Where the 8-character hash is the first 8 characters of the hash of the section discriminator. For example:

```
integrations:keyspaces:abc12345:fsccorporate
```

[Erik Andersson]: The purpose of this hash is purely for **uniqueness across multiple installations of the same CRM system**. Since you can have multiple installations of a CRM system on different sections or even reinstalled multiple times on the same section, we needed something unique to identify this specific instance.

[Lukasz Grabowski]: So this is only for uniqueness, not for identifying the section in reverse—we don't need to resolve which section this keyspace belongs to based on the hash alone?

[Erik Andersson]: Correct, it's just for uniqueness. I don't know why we specifically chose 8 characters, but it was a deliberate choice. The important thing is that it ensures each installation gets a unique discriminator.

### Keyspace Retention on Uninstallation

A critical design decision: **keyspaces are never deleted when an installation is uninstalled**.

[Lukasz Grabowski]: When we uninstall, do we delete the keyspace?

[Erik Andersson]: No, we don't delete keyspaces. When you reinstall the same CRM system on the same section, we adopt the existing keyspace.

**Rationale for retention**: Even if technically possible to delete keyspaces, there was a historical requirement that profile sync and access data should remain in Apsis unless explicitly removed by someone performing GDPR cleanup. The business logic is:

- Customers may be won back, and their historical data should be preserved
- For **contacts**, you can always re-sync from the CRM, but for **historical event data**, this information would be permanently lost if the keyspace were deleted
- This design prevents data loss for events and historical interactions that are immutable

---

## Installation Flow and Keyspace Creation

### Main Keyspace vs. Entity Keyspaces

The installation process (handled in the **installation manager's main install function**) creates two types of keyspaces:

1. **Main keyspace**: Created for the primary contact entity, using just the configuration
2. **Entity keyspaces**: Created for additional entity types beyond the standard contact (leads, silhouettes, etc.)

```
Installation Manager Flow:
├── Create main keyspace (contact/person)
│   └── Uses: configuration
└── For loop: additional entities
    └── Create entity-specific keyspaces (leads, silhouettes)
        └── Uses: entity-specific discriminators
```

[Erik Andersson]: We generate these keyspace discriminators at installation time and store them in the database. When we need to interact with a specific entity type, we do a lookup: "Give me the keyspace ID with this discriminator," then use that ID for profile updates.

---

## Lead and Silhouette Keyspaces: Why They're Necessary

### The Problem They Solve

**Lead/silhouette keyspaces exist to keep certain entity types separate from full contacts** until an explicit merge occurs.

[Erik Andersson]: For example, with E-deal, we will not download any leads into Apsis. However, E-deal will respond with a silhouette ID from a form submission. We cannot add that silhouette ID to the normal (contact) keyspace because:

1. It would be interpreted as a full contact
2. If we later do a consent export back to the CRM, we would find that entry and send it
3. This creates confusion—contacts and leads should remain separate until there's an explicit merge request

[Lukasz Grabowski]: So you mean the CRM-created keyspace that we discussed earlier—we cannot use that for silhouettes?

[Erik Andersson]: Exactly. If we did, we'd have issues especially with consents. When a CRM ID has a consent change, our listeners would pick it up and start sending consent changes for silhouettes back to Apsis—but that's not part of the scope. We should only:
- Send form submissions to the CRM
- Register the silhouette IDs on the profile

### Entity Keyspace Discriminator Structure

For entity-specific keyspaces (e.g., silhouettes), the discriminator extends the main format:

```
integrations:keyspaces:<8-char-hash>:<crm-logical-name>:<entity-name>
```

For example:
```
integrations:keyspaces:abc12345:fsccorporate:silhouettes
```

[Erik Andersson]: We considered adding even more specificity (e.g., `.contact` to denote the entity type), but that would have required significant data migration for existing customers, so we chose not to.

---

## Keyspace Lookup and Profile Update Flow

### How Keyspace IDs Are Retrieved and Used

When interacting with CRM entities, the system follows this pattern:

1. **Know what entity type you're working with** (e.g., you downloaded "persons" from the CRM)
2. **Look up the keyspace discriminator** for that entity type
3. **Retrieve the keyspace ID** from the database using that discriminator
4. **Use that keyspace ID for profile updates**

[Erik Andersson]: For example, when we download entities from the CRM, they tell us "here are all persons." We know we're dealing with persons, so we look up the keyspace discriminator for persons, retrieve its ID, and use that for all subsequent updates.

### Processing Form Submissions with Multiple Entity Types

When a form is submitted and the CRM responds:

1. The CRM informs us what entity type was created (contact, silhouette, lead, etc.)
2. We check our mappings: "For a silhouette, what is the keyspace discriminator?"
3. We retrieve the corresponding keyspace ID
4. We update the profile with the returned entity ID using that keyspace

---

## CRM-Specific Entity Behavior

### E-deal: Contacts and Silhouettes

**Keyspaces created on installation**:
- Main contact keyspace
- Silhouette keyspace

**Form submission behavior**: E-deal responds with either:
- A new **silhouette** (lead-like entity)
- An existing contact (if it matches)

**Merge scenarios**: E-deal allows customers to later merge silhouettes into contacts:
- Customer submits form → silhouette created
- Later: "We met with this lead and have all their info" → merge request sent
- Apsis receives merge request, merges silhouette and contact profiles using their respective keyspaces

---

### FSC Corporate: Contacts and Silhouettes

Same pattern as E-deal: creates both contact and silhouette keyspaces, responds with either entity type on form submission.

---

### FSC Enterprise 12.1: Contacts Only

[Erik Andersson]: FSC Enterprise 12.1 doesn't have a concept of leads or silhouettes. They always create contacts. When you install FSC Enterprise 12.1, you will only see the main FSC Enterprise 12.1 keyspace—that's the only thing we'll ever use.

**Form submission behavior**: Always responds with contacts
**Download behavior**: Always returns contacts
**Webhook behavior**: Always sends contact data

This simplifies the integration but requires FSC Enterprise to separate temporary and permanent contacts on their side—not Apsis's concern.

---

### Tribe: Dynamic Entity Selection

**Keyspaces created on installation**:
- Main contact keyspace
- Lead keyspace (bootstrapped even though it's not fully used)

**The complication**: Tribe has a virtual "lead" concept with a dynamic dropdown allowing customers to select which entity type gets created from form submissions (contacts, leads, or various other custom entities).

[Erik Andersson]: The problem is that Tribe allows selecting from 10+ different entities, but we don't support completely dynamic entities in Apsis. We need to know:
- What is the unique field for this entity?
- What is the name of this entity?

When we ask Tribe "give me the fields for this lead," they return the fields for whatever entity is currently selected in their dropdown. If they've selected "contacts," we get contact fields. If they've selected "sailing boats," we get sailing boat data.

[Lukasz Grabowski]: That's very complicated.

[Erik Andersson]: Yes, it's irritating. Our lead entity in Apsis essentially represents their dynamic dropdown. I'm trying to simplify it for them, but that's how it is currently.

---

### Dynamics and Other CRM Systems

**Legacy Dynamics**: Contact keyspace only

**Dynamics via Site Shop**: Both contact and lead keyspaces (supports leads)

**Other systems**: Similar patterns to those already described

---

## Form Submission Event Handling

### Event Listener Registration

Form submission events are only processed if:
1. The CRM integration is installed on the section
2. The form is explicitly configured to "sync to CRM"
3. The CRM system supports sending form submissions

[Erik Andersson]: You can't even select "sync to CRM" unless the integration is installed. When you do enable it, we register event listeners for all possible campaign events. For form campaigns, we listen for **start**, **viewed**, and **submit** events.

### The Form Submission Flow

When someone submits a form:

```
User submits form
  ↓
Form data collected (email, first name, last name, custom fields, profile key, etc.)
  ↓
Submit event sent to integration
  ↓
Integration forwards to CRM system with:
  - profile key (from Apsis)
  - identifying fields (email, phone, etc.)
  - custom form fields
  - campaign/event context
  ↓
CRM responds with created entity type and ID
  ↓
Apsis maps response to appropriate keyspace and merges/stores ID
```

### Identifying Information in Form Submissions

The CRM system uses these to identify or create contacts:
- Email address
- Phone number
- Existing CRM ID (if pre-filled, which became possible with pre-filled forms)

[Erik Andersson]: We send identifying data under a special property called `fields` so the CRM knows what to match on.

### No Events for Unsupported Systems

[Erik Andersson]: If a CRM system doesn't support sending form submissions, `can_sync_form_activities` is set to `false`. This means:
- The "sync to CRM" option is **not displayed** in the form UI
- When the form is published, **no request is sent to integration**
- **No event listeners are registered**
- Even if someone somehow submitted a form, we would never receive the event for that installation

This guarantees **under no normal circumstances will we get form submit events for integrations that either aren't installed or don't support the feature**.

---

## Profile Merging: A Critical Boundary

### The Core Rule: Never Merge Across Keyspaces

[Erik Andersson]: In integration, we never merge with any keyspaces apart from merging a lead keyspace with its corresponding contact keyspace. We will never merge with the email keyspace or anything else.

### Why This Matters

[Erik Andersson]: The CRM system should be considered the master of the data. If you merge two profiles—say a profile from the email keyspace with a CRM contact profile—you might update attributes from one with data from the other. For example:
- Email keyspace profile has first name: "John"
- CRM profile has first name: "Jane"
- Merge happens, now one data source has "Jane"
- But the CRM system wasn't involved in this merge, so it's now out of sync
- If you send an email to this person, you risk adding completely different personal data than they actually have

This is data integrity risk that outweighs any benefit from merging.

### Merging Must Originate from the CRM

[Erik Andersson]: If contacts from the CRM need to be merged, they should be merged **from within the CRM system** because:
1. That merge will reach Apsis (through webhooks or API calls)
2. The CRM system remains the authoritative source
3. When merges happen inside Apsis but originate from other tools, **they are not propagated back to the CRM**, causing data drift

If someone merges two CRM contacts in Apsis but the CRM doesn't know about it, the CRM will continue treating them as separate entities. You now have a dangerous inconsistency.

### Current Practice

[Erik Andersson]: We allow merging between lead and contact keyspaces **within the same CRM installation** (e.g., silhouette + contact for E-deal), because this merge is initiated by the CRM itself sending us a merge request. This maintains consistency.

---

## Unresolved Questions and Follow-up Items

### Custom Keyspace Configuration Request (Deferred)

A customer using the **generic/joined CX connector** has requested the ability to override the default keyspace behavior and use the **email keyspace** instead of the CRM ID keyspace for profile identification. 

[Erik Andersson]: They're using a custom integration outside of our standard ones and prefer email-based matching to CRM IDs. They want us to configure the generic connector to use the email keyspace for their installation.

**Concerns raised**:
- If implemented in the generic connector, this would affect **all joined CX customers**, not just this one
- No clear business case if it's a single customer request
- Would require architectural changes that both speakers want to avoid

[Lukasz Grabowski]: This is not in our design, not in our resources, and not in our interest to handle. This is a really custom thing for one customer, and the chance we'll do anything about it is very small.

[Erik Andersson]: There may be ways to make this happen without coding, but it needs more focused discussion time.

**Status**: Deferred to following week (mid-week meeting tentatively scheduled). Erik to reply to customer communication indicating the team will investigate further but needs more time. Both speakers indicated this is a low-priority item unlikely to proceed.

---

## Key Takeaways

1. **Keyspace discriminators are unique per CRM installation**, constructed from a hash of the section identifier and the CRM's logical name. They are used purely for uniqueness, not for reverse lookups.

2. **Keyspaces are never deleted** to preserve historical event data and enable customer win-back scenarios, even when integrations are uninstalled.

3. **Entity keyspaces (leads, silhouettes)** are essential for keeping different entity types separate until explicitly merged by the CRM system itself.

4. **Lead/silhouette keyspaces prevent data integrity issues** by ensuring that temporary or prospect data isn't mistakenly treated as full contact records, especially for consent management.

5. **CRM systems have varying entity models**: some support leads (E-deal, Tribe, Dynamics via Site Shop), some don't (FSC Enterprise 12.1), and some are completely custom (Tribe's dynamic dropdown).

6. **Form submission events are only processed** if the integration is installed and explicitly enabled on that section for that form.

7. **Merging profiles across keyspaces is a critical boundary** that must not be crossed except within a single CRM installation's own entity types. Cross-keyspace merges risk data integrity because they break the CRM system's authority as the data master.

8. **Merges should always originate from the CRM system** to ensure consistency and proper propagation back to the source system.

9. **Tribe's integration is particularly complex** because its entity type selection is dynamic—the "lead" keyspace represents their dropdown choice, not a true lead entity.

---

## Unresolved Questions

- **Custom keyspace override request**: How to handle a customer's request to use the email keyspace instead of CRM IDs in the generic connector—deferred to mid-week meeting for further investigation.