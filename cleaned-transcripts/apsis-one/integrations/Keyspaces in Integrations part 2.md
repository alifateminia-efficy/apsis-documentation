---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [Keyspace architecture and design, Entity-specific keyspaces, CRM installation flow, Form submission handling, Lead vs contact entities, Data merging and duplication prevention, Integration-specific behaviors]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace discriminators, Installation manager, Entity keyspace mapping, Form sync to CRM, Silhouette entities, Profile merging, Event listeners]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.0, Efficy Enterprise 12.1, E-deal (Efficy Corporate), Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This session covers the second part of a detailed technical discussion on keyspace implementation in the Apsis One Integrations domain. The conversation focuses on how keyspaces are designed and used across different CRM systems, the distinction between main contact keyspaces and entity-specific keyspaces (particularly for leads and silhouettes), the mechanics of form submission handling, and the nuances of how different CRM systems (E-deal, Efficy Enterprise 12.1, Tribe, Microsoft Dynamics, etc.) handle entity creation differently. The session also touches on a pending customer request regarding custom keyspace behavior that the team is deferring for further investigation.

## Keyspace Design and Discriminator Structure

### The Hash-Based Discriminator Pattern

The keyspace discriminator follows a consistent pattern across all CRM installations:

```
integrations.keyspaces.<hash>.<crm_logical_name>
```

Where `<hash>` is the first 8 characters of a hash derived from the section discriminator.

[Erik Andersson]: > The discriminator here is like `call_matches_1_integrations_keyspaces_[random_characters]`, but these characters are the hash of the section discriminator and then we add the logical name of the CRM.

### Purpose of the Hash: Uniqueness, Not Reversibility

[Lukasz Grabowski]: The initial question was about the purpose of including a hash in the discriminator. Is it for uniqueness, or for identifying which section a hash belongs to?

[Erik Andersson]: It's purely for uniqueness. The reason is that you can have multiple installations of a CRM system on the same section. We needed something unique for each specific installation.

[Lukasz Grabowski] clarified an important point: the hash is not designed to be reversible or to allow reverse lookup from the hash back to the section. It serves only as a collision prevention mechanism.

[Erik Andersson]: The specific choice of 8 characters was somewhat arbitrary—a design decision made at the time without strong technical justification beyond providing sufficient uniqueness.

### Lifecycle: No Deletion on Uninstall

When an integration is uninstalled, the keyspace is **not deleted**. This is a deliberate design choice with important business implications:

[Lukasz Grabowski]: If we uninstall, do we delete the keyspace?

[Erik Andersson]: No, we don't delete keyspaces. If you create the same installation again on the same section, we adopt the existing keyspace.

**Rationale for preservation**: There are regulatory and business continuity requirements:
- Profile sync data must remain in Apsis unless explicitly deleted (e.g., via GDPR cleanup request)
- This allows customers to win back profiles if they re-establish a relationship with the CRM
- Historical event data is particularly valuable and cannot be regenerated once deleted
- If a customer uninstalls and reinstalls the same integration, their historical data remains available

## Main Keyspace vs. Entity-Specific Keyspaces

### The Installation Flow

The installation process creates multiple keyspaces depending on the CRM system type. This is handled in the **installation manager's main install function**:

1. **Main keyspace** is set up first using the configuration
2. **Additional entity keyspaces** are created via a loop for entities beyond the primary contact

### Why Separate Keyspaces for Leads/Silhouettes?

The clearest example is E-deal (Efficy Corporate), which distinguishes between leads and contacts:

[Erik Andersson]: For E-deal we need to create a separate keyspace for silhouettes (which E-deal calls their leads). While we don't download leads from E-deal directly, E-deal responds with a **silhouette ID** from form submissions. We cannot add that silhouette ID to the normal contact keyspace because:

1. It would be interpreted as a full contact
2. When we export consent changes back to the CRM, we would find that entry
3. Leads and contacts should remain separate until an explicit merge request is sent from the CRM

[Lukasz Grabowski]: You're referring to the CRM-created keyspace that we discussed—the dedicated installation keyspace. We can't use this for silhouettes?

[Erik Andersson]: Exactly. If we stored silhouette IDs in the main contact keyspace and a consent change occurred:
- Our listeners would pick up the change
- We would send consent updates back to the CRM for the silhouette ID
- This is out of scope—we should only send form submissions and register silhouette IDs

### Keyspace Discriminator for Entities

The entity keyspace discriminator extends the main discriminator with the entity name:

```
integrations.keyspaces.<hash>.<crm_logical_name>.<entity_name>
```

For example, for E-deal silhouettes:

```
integrations.keyspaces.<hash>.e_deal_corporate.silhouette
```

[Erik Andersson]: We considered adding `.contact` to be even more specific (e.g., `.silhouette.contact`), but this would have required significant data migration for existing customers, so we chose the simpler format.

### How Keyspace Mappings Are Stored and Retrieved

The system stores mappings in the database at installation time:

1. At installation: Generate the keyspace discriminator for each entity and store it
2. At runtime: Look up the keyspace ID by discriminator when needed
3. When downloading from CRM: Query "give me all persons" → know we're working with the person entity → retrieve the person keyspace ID → use it for profile updates
4. When processing CRM responses: CRM says "this ID is a silhouette entity" → check mapping → retrieve silhouette keyspace ID → use it for updates

[Erik Andersson]: When I download entities from the CRM system, we download all persons, then we know exactly which keyspace we need. We do a lookup and retrieve the keyspace ID, then utilize that ID when we do updates.

## Form Submission Flow and Entity Creation

### Event Registration and Sync Conditions

Form sync to CRM only works when:
1. The integration is installed on the section
2. The CRM system supports syncing form activities (not all do)

[Erik Andersson]: You can't even select "sync to CRM" unless the integration is installed. When you publish a form with sync enabled, event listeners are registered for `start_viewed` and `submit` events.

[Lukasz Grabowski] noted that if a CRM is installed but `can_sync_form_activities` is `false` (e.g., Microsoft Dynamics doesn't support this), the sync option won't even display, and no event listeners will be registered.

### Form Submission Request Structure

When a form is submitted, the system sends:

```
{
  "campaign_events": {
    "points": [
      {
        "profile_key": "<apsis_profile_key>",
        "fields": {
          "email": "<submitted_email>",
          "first_name": "<submitted_first_name>",
          "last_name": "<submitted_last_name>",
          "custom_field_1": "<value>"
        },
        "existing_identifiers": {
          // if the profile has a CRM ID or Lead ID
          "crm_id": "<id>",
          "lead_id": "<id>"
        }
      }
    ]
  }
}
```

The **profile_key** is always included. Identifying data (email, phone, CRM ID, Lead ID) is included in `fields` if available.

[Erik Andersson]: Historically, there was no way for the CRM to identify an existing contact using a CRM ID unless the form was pre-filled. With the new pre-filled form feature, you can now send a CRM ID in the form submission, which the CRM can use for identification.

### CRM Response Handling: New vs. Existing Records

The CRM responds with information about what record was created or matched:

```
{
  "entity_type": "contact" | "silhouette" | "lead",
  "record_id": "<new_record_id>"
}
```

The CRM can respond with:
1. **A new record**: "We created a new record of type [contact|silhouette|lead] with ID X"
2. **An existing record**: "We already have a [contact|silhouette|lead] with this email/phone/ID; we matched to ID X"

### Linking and Merging with the Silhouette Keyspace

When a silhouette (lead) is returned, the system:

1. Knows which profile submitted the form (via profile_key)
2. Stores the silhouette ID as a silhouette identifier for that profile
3. **Merges** the export keyspace (profile's main keyspace) with the silhouette keyspace

[Erik Andersson]: We tie those two together using a merge between the export keyspace with the profile_key and our silhouette keyspace with the returned ID. This prevents creating duplicates if the same person submits the form multiple times.

## Entity Handling Across Different CRM Systems

### E-deal (Efficy Corporate)

- **Creates both contact and silhouette keyspaces** at installation
- **Supports silhouettes** (their concept of leads)
- Responds to form submissions with silhouette entities
- Responses may indicate existing customers as contacts or new leads as silhouettes

### Efficy Enterprise 12.1 (Generic Connector)

- **No concept of leads**—only contacts
- **Creates only the main contact keyspace** at installation
- Always creates or returns contacts (never silhouettes)
- Simpler behavior but customers must separate temporary and permanent contacts themselves

[Erik Andersson]: When you install FEC Enterprise 12.1, you'll only see the FEC Enterprise 12.1 keyspace. They respond with contacts for form submissions, downloads, and webhooks. It's nice in that it's simpler, though it means they have to manage temporary vs. actual contacts internally.

### Tribe

- **Has a lead entity in theory**, but implementation is problematic
- **Creates a lead keyspace** at installation
- **Never responds with leads** from form submissions
- Their lead is virtual—form submission results depend on a **dynamic dropdown** in Tribe where the customer selects which entity type to create

[Erik Andersson]: This is super irritating. When we ask Tribe for the fields available for "lead", they return the fields for whatever is currently selected in their dropdown. If they've selected "contact", we get contact fields. If they've selected "sailing boat" (a custom entity), we get sailing boat fields. Our lead keyspace in Apsis is essentially a placeholder for their dynamic dropdown behavior. I'm trying to simplify this for them.

### Microsoft Dynamics (Multiple Versions)

**Legacy Dynamics**:
- Only contact entity
- No lead support
- Creates only the main contact keyspace

**Dynamics CRM by Site Shop**:
- Supports leads
- Creates both contact and lead keyspaces at installation

**Dynamics (365/Online)**:
- Only contact entity
- Same as legacy

### Summary Table of Entity Support

| CRM System | Contact Keyspace | Lead/Silhouette Keyspace | Form Response Type |
|---|---|---|---|
| E-deal (Efficy Corporate) | ✓ | ✓ (silhouette) | Contact or silhouette |
| Efficy Enterprise 12.1 | ✓ | ✗ | Contact only |
| Efficy Enterprise 12.0 | ✓ | ✗ (legacy) | Contact only |
| Tribe | ✓ | ✓ (placeholder) | Dynamic (contact, or custom) |
| Microsoft Dynamics (legacy) | ✓ | ✗ | Contact only |
| Microsoft Dynamics by Site Shop | ✓ | ✓ (lead) | Contact or lead |

## Profile Merging and Data Integrity

### Why Avoiding Merges with Other Keyspaces Matters

The integration system intentionally **does not merge** CRM contact keyspaces with email keyspaces or any other keyspaces. This was a deliberate design change:

[Erik Andersson]: We used to merge CRM contacts with the email keyspace, but product requested we remove that because there can be multiple contacts in the CRM with the same email address. If we merge on email, that possibility disappears.

### The CRM as Master Data

[Erik Andersson]: The CRM system should be considered the master of the data. If you merge a CRM contact with a non-CRM profile (e.g., email keyspace profile) and update attributes, the CRM data will be considered out of sync. If you then send an email to that person, you risk including completely different personal data than what they actually have in the CRM. This is sensitive and not super good.

### When and Where Merging Can Happen

**Allowed within integrations**:
- Lead keyspace merges with contact keyspace (when a lead becomes a contact)
- This happens via explicit merge requests from the CRM

**Never allowed**:
- CRM contact keyspace merging with email keyspace
- CRM contact keyspace merging with any other external keyspace

[Lukasz Grabowski] asked: Can we merge two profiles where one has a CRM ID and the other has the same email?

[Erik Andersson]: Not in integration. We never merge with any keyspaces apart from within our own (e.g., lead + contact). This can happen from inside Apsis itself, but it creates weird behavior, which is why we're not fond of other tools manipulating contacts that come from a CRM system unless those merge requests originate from the CRM.

### Merging Should Happen in the CRM

[Erik Andersson's opinion]: If contacts that come from a CRM need to be merged, they should be merged in the CRM itself. When you merge them in the CRM, that change reaches us and reaches Apsis. If you merge them inside Apsis, that change is never proliferated back to the CRM, leading to data inconsistency.

## Event Listener Lifecycle

Event listeners are registered when an integration is installed and has form sync enabled. They are removed when:
- The integration is uninstalled
- The form sync is disabled for a campaign

[Erik Andersson]: Under no normal circumstances will we get actual form submit events to an installation that either isn't installed or doesn't support form sync.

## Pending Customer Request: Custom Keyspace Override

### The Request

A customer (using a custom integration outside the standard Apsis connectors) is asking for the ability to use the **email keyspace** instead of the default CRM keyspace for contact identification. They are not interested in CRM IDs and want to work entirely with email addresses.

[Erik Andersson]: They want us to generate the email keyspace discriminator instead of the default joined CX keyspace discriminator. They'd want to override the default behavior in the generic connector to say "use the email keyspace instead."

### Issues with the Request

1. **Scope problem**: If we implement this, it would apply to **all** joined CX customers, not just this one customer
2. **Implementation complexity**: Very complicated to add a custom override mechanism
3. **Weak business case**: Only one customer has requested it
4. **Design mismatch**: The keyspace architecture is fundamentally designed around CRM IDs as the primary identifier

[Erik Andersson]: I see absolutely no business case for this if it's just one customer asking.

### Decision: Deferred Investigation

The team decided to defer this request:
- They will respond to the customer that more time is needed for investigation
- Lukasz will be on leave for three days
- A follow-up meeting is scheduled for **Wednesday of the following week** to discuss potential solutions
- No code changes will be made without further analysis
- [Erik Andersson]: There are ways to solve this without coding, but it needs more focused discussion time

---

## Key Takeaways

1. **Keyspace discriminators use a hash + logical name pattern** for uniqueness but are not designed to be reversible. The hash is arbitrary (8 characters chosen for collision prevention).

2. **Keyspaces are never deleted on uninstall**—they're preserved to maintain historical data and allow customer reactivation, with explicit GDPR cleanup being the exception.

3. **Different CRM systems have different entity models**:
   - E-deal has contacts and silhouettes
   - Efficy Enterprise 12.1 has only contacts
   - Tribe has a dynamic entity dropdown that complicates lead handling
   - Microsoft Dynamics variants have different levels of lead support

4. **Lead/silhouette keyspaces exist to prevent data corruption**—without them, lead submissions would be interpreted as contacts, and consent updates would leak into out-of-scope CRM operations.

5. **Form submissions require identifying information** (email, phone, CRM ID, or lead ID) and use the profile_key to link responses back to the submitting profile. Pre-filled forms enable CRM ID submission.

6. **The CRM is the master of truth** for contact data. Merging CRM profiles with non-CRM profiles (like email keyspace) creates sync issues and data inconsistency. Merges should happen in the CRM and flow back to Apsis, not the reverse.

7. **Event listeners are tied to installation state**—they're only registered when the integration is installed and supports form sync, and they're removed on uninstall or sync disable.

8. **The pending JoinCX custom keyspace request is deferred** due to scope concerns (would affect all JoinCX customers), weak business justification (one customer), and implementation complexity. The team will explore non-code solutions after Lukasz returns from leave.

---

## Unresolved Questions and Action Items

- **Pending**: Customer request regarding email keyspace override for JoinCX integration. Follow-up meeting scheduled for mid-week (Wednesday) next week to discuss potential solutions that don't require code changes.
- **Ownership**: Erik Andersson will reply to the pending email thread in Swedish to acknowledge the request and set expectations for further discussion.