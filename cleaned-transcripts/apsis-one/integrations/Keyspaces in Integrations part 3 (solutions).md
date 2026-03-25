---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One Integrations
topics: 
  - Keyspace configuration and profile management
  - Multi-identifier challenges in integrations
  - Profile merging and deduplication strategies
  - Join CX integration with email-based identifiers
session_type: architecture-review
speakers: 
  - Erik Andersson (Apsis platform expert)
  - Lukasz Grabowski (Integration specialist)
key_components: 
  - Keyspaces (CRM ID keyspace, Email keyspace, Join CX keyspace)
  - Profile merge worker service
  - Integration configuration system
  - Form submission handling
  - Consent/subscription updates
subdomains: 
  - Lead creation
  - Duplicate profiles in Apsis
---

## Session Overview

This session discusses a complex integration scenario where an end customer uses **Join CX** (a customer loyalty system by Intermail) as an intermediary that syncs customer data to **Apsis One**. The core problem: Join CX operates using **CRM IDs** as identifiers and syncs to Apsis using the Join CX keyspace, but the end customer's own API operations use **email addresses** as identifiers and expect to work with the email keyspace. This mismatch creates duplicate profiles and data retrieval failures. The team evaluates four potential solutions ranging from full system redesign to pragmatic workarounds, ultimately recommending a hybrid approach combining database reconfiguration with customer-side changes.

---

## Problem Statement: The Identifier Mismatch

### The Integration Architecture

The setup involves three parties:

1. **End Customer**: Operates their own system, communicates with Apsis via API, uses **email addresses** as their primary identifier
2. **Intermail (Join CX)**: Operates a customer loyalty system, integrates with end customer's system, syncs data to Apsis using **CRM ID** as the identifier
3. **Apsis One**: Receives data from both Join CX (via CRM ID/Join CX keyspace) and directly from the end customer's API (expecting email keyspace)

[Erik Andersson]: "The customer is integrated with their system and here they are, they are only using email addresses when they interact with their system... Join CX will sync things to Apsis. But because this is general connector, they are using their CRM ID identifier... the customer has no idea what the CRM ID is. Their customer they only know using their email."

### The Critical Issue

The customer wants to utilize the **same profiles** that were created via Join CX (in the CRM ID keyspace) but access them using **email addresses** through the email keyspace. This is not a typical use case and creates several problems:

**Data Retrieval Failure**: When the end customer queries for a profile by email address, no matching profile is found (because profiles are keyed by CRM ID in the Join CX keyspace).

**Duplicate Profile Creation**: If the end customer tries to update a profile via email, the system creates a new profile in the email keyspace rather than updating the existing CRM-keyed profile.

**Event Data Loss**: Custom events added via Join CX (e.g., loyalty points) are attached to the CRM ID profile and become unreachable when querying by email.

[Lukasz Grabowski]: "So for them if they could get for example profile by email attribute... they then that that would work I guess right? But of course we are talking about one, so it's it's not possible because they want to get use profile through the key space and through the profile key."

[Erik Andersson]: "If they would do as you said they want to update like add a secondary mobile number like what would happen is now there will have a second duplicate profile in the email via the email key space instead of the CRM key space. So whatever their use case, it will either create duplicates or return no matching data."

---

## Solution Analysis: Four Approaches

### Solution 1: Full Keyspace Configuration (Most Complex)

**Concept**: Allow customers to specify which keyspace an integration should use during installation via a dropdown UI. The generated connector code would then use the selected keyspace (email vs. CRM ID) as the primary identifier throughout all operations.

**Implementation Requirements**:

- Add keyspace selection to the installation UI with options for each available keyspace
- Pass the selected keyspace to the connector code generator
- Implement conditional logic throughout the codebase to extract and validate the correct identifier type

**Critical Constraints**:

The identifier extraction logic must change based on keyspace selection. For the CRM ID keyspace, any string value can be used. For the email keyspace, the value **must be a valid email address**. This means:

- Code cannot extract a contact ID and treat it as the identifier
- Code must validate email format before using as an identifier
- Consent/subscription updates would need to send email address in the `record_id` field instead of CRM ID

[Erik Andersson]: "The unique identifier would also need to be configured for it because for the CRM ID for the CRM key space like you can specify essentially anything... but for the email key space it must be a valid email address... we need to download contacts and extract the email address... use that as the unique identifier and this is also not something which exists today."

**Mutual Exclusivity Constraint**: 

The customer cannot support multiple profiles with the same email address if using the email keyspace. This is a fundamental constraint of using email as a unique key.

[Lukasz Grabowski]: "If there is a key, it's an email. So it's just in one email for one person. It cannot be like multiple... it must be unique."

**Effort Estimate**: High (multiple weeks, requiring significant code changes)

**Assessment**: Erik explicitly states, "I am not really suggesting that we go with this approach. I'm just lifting it to show you what's like there the different objects there are."

---

### Solution 2: Automatic Profile Merging (Moderate Complexity)

**Concept**: Keep the current keyspace configuration (CRM ID/Join CX keyspace), but automatically trigger a **profile merge operation** after each profile update. This merges the CRM-keyed profile with an email-keyed profile, allowing the same customer record to be found via either identifier.

**How the Merge Service Works**:

The profile merge worker service (already exists in the platform) is currently used primarily for form submission handling. It takes:
- A **source keyspace** and **source key** (e.g., Join CX keyspace, CRM ID "12345")
- A **destination keyspace** and **destination key** (e.g., Email keyspace, "customer@example.com")

The service merges these into a single unified profile.

[Erik Andersson]: "So we merge this profile that exists right now, like in your form or email key space. We merge that with our silhouette key space for Edeal or in the case of the FC Enterprise to us... so the profile can be found or like so we don't create duplicates."

**Implementation Approach**:

1. After creating/updating a profile in the CRM ID keyspace, automatically queue a merge message to the integration profile merge worker service
2. Extract the email address from the profile attributes
3. Send merge request with:
   - Source keyspace ID: Join CX keyspace ID
   - Source key: CRM ID
   - Destination keyspace ID: Email keyspace ID
   - Destination key: email address (extracted from profile)

**For First Iteration**: The email keyspace ID can be **hardcoded** since this is a customer-specific implementation.

[Lukasz Grabowski]: "We can hard code it his ID like it will be very not elegant, elegant solution, but we can do do them for some time at least."

[Erik Andersson]: "We do have a merge worker in integration already... it would not be a big thing for you to extract the ID for the email key space and add to that as a step in the profile update service. That would take you like 2-3 days."

**Profile Creation Behavior**:

If a profile doesn't exist in the email keyspace yet, the merge service will create an empty profile and then merge it into the source profile. This is not a problem.

[Lukasz Grabowski]: "But I mean, merge. You need to merge to something. I mean if we don't have profile in the email key space."

[Erik Andersson]: "If there it doesn't exist, I think they create an empty profile. So that's that's not that's not a problem."

**Effort Estimate**: 2-3 days (moderate, leverages existing merge functionality)

**Advantages**:
- Requires no changes to data model
- Reuses proven merge worker infrastructure
- Could be extended to other customers in the future
- Leaves system in "intended" state (profiles remain in primary CRM ID keyspace, email becomes secondary access path)

**Disadvantages**:
- Still requires action items on the Apsis side
- Subject to product roadmap and business priorities
- Would likely be completed in 3-4 months at current velocity

---

### Solution 3: Database Configuration Hack (Fastest, Customer-Dependent)

**Concept**: Bypass code changes entirely. Directly modify the **integration mapping table** in the database to point the integration's entity type (e.g., "Member") to the email keyspace instead of the Join CX keyspace. Whenever the integration processes updates for that entity type, it will write to the email keyspace using the mapping.

**How It Works**:

Apsis maintains a mapping table for each integration installation that specifies which keyspace to use for each entity type. For example:

```
integration_id: join_cx_prod
entity_type: member
keyspace_id: join_cx_keyspace_id
```

This mapping controls all profile updates. Instead of changing code, simply update the database:

```
keyspace_id: email_keyspace_id  // changed from join_cx_keyspace_id
```

[Erik Andersson]: "We the customer installs as normal, but instead of it saying like this member key space here, we would just go into the database and change this to the email key space then the integration. Whenever there is an update for member, it will try and do those updates on the email key space instead and this is without changing any code or logic on Apsis side, this will take you 5 seconds to change and then everything in Apsis is fixed essentially."

**Critical Requirement: CRM ID Must Be Email Address**

This is where the customer's action is critical. The integration still uses the **CRM ID field** as the profile identifier. For this to work with the email keyspace, the customer must:

1. **Change their API calls** to send the customer's email address in the `CRM_ID` field (instead of the actual CRM ID value)
2. **For consent/subscription updates**, send the email address in the `record_id` field instead of the CRM ID
3. **For all data syncs** (initial and ongoing), ensure email address is provided in the CRM ID position

[Erik Andersson]: "We will still look for the CRM ID as the identifier, but as we mentioned in the beginning, this will not work if we try to update the email key space... We would need the email addresses provided to us as the CRM ID, meaning this solution will be a workaround."

[Lukasz Grabowski]: "We need to synchronize the change right? If we say OK, please send us a CRM email address in CRM ID and then let us know when it's done then we will switch the database and that's it."

**Effort Estimate**: 
- Apsis side: 5 minutes (literally a database UPDATE statement)
- Customer side: Minimal (API field mapping change)

**Advantages**:
- Immediate implementation possible
- No code changes required on Apsis side
- Customer can execute their changes independently without waiting for Apsis roadmap
- Can be tested and deployed within days

**Disadvantages**:
- **Critical caveat**: If the customer reinstalls the integration, the database change reverts to default (CRM ID keyspace) and breaks the integration. Manual intervention required again.
- Creates technical debt (hardcoded mapping in database)
- Not scalable as a general solution
- Relies entirely on customer's continued cooperation

[Erik Andersson]: "The caveat here is of course if they for whatever reason were to reinstall the installation, then this change must again be done on the side because we will have reverted to the normal joint CX in the database because that's set up on installation."

**Risk: Existing Profiles**

If the customer already has synced profiles in the CRM ID keyspace, switching the mapping will cause **duplication**: old profiles remain in CRM ID keyspace, new updates go to email keyspace. Mitigation options:

1. Customer performs full manual merge of old profiles
2. Customer deletes all existing profiles before switch
3. Accept duplication as a legacy data issue

[Erik Andersson]: "If they already have existing contacts, they would exist of course in the join CX key space. If we change which key space, everything would be duplicated unless they do manual merge work before or they just delete every profile they have."

---

### Solution 4: Do Nothing - Follow Original Design (No Development)

**Concept**: Decline the feature request. The integration was designed to use CRM IDs as the primary identifier. The customer should either:

1. Maintain their integration through Join CX using CRM IDs (the designed path)
2. Do the translation work themselves between their email-based API and the CRM ID-based Apsis integration

[Erik Andersson]: "The customer utilizes the CRM ID and the join CX key space in their custom solution is that meaning they would need to essentially actually do the work between the join CX instance instead."

[Lukasz Grabowski]: "The system this integration one was designed to rely on CRM ID and not email, right?"

**Effort Estimate**: 0 on Apsis side

**Assessment**: Likely not acceptable to the customer, but worth mentioning as an option.

---

## Recommendation Summary

**Erik's Position**: "If anything had to be done on Apsis side, I would go for the merge one."

**Lukasz's Recommendation**: Solution 3 (database hack) is most practical for this single customer, exceptional case scenario. The trade-offs are acceptable given:
- Limited scope (one customer)
- Non-standard use case
- Ability for customer to execute independently
- Risk can be managed through documentation

**Next Steps**: 
- Present Solutions 2 and 3 to product leadership (Alexander Linzkok, Opri)
- Do NOT present Solution 1 (too complex)
- Mention Solution 4 as fallback (follow original design)
- Let Alexander and product team evaluate priority against other work

---

## Historical Context: Why Profile Merge Exists

The merge worker service exists because form submissions create a particular problem:

1. Customer fills out a form in Apsis (creates profile in form/email keyspace using email)
2. Form triggers an event sent to the CRM system (Join CX, Efficy, etc.)
3. CRM system creates a new entity (e.g., a Lead) and returns a CRM ID
4. Apsis needs to link the original form profile (email-keyed) with the CRM entity (ID-keyed) to prevent duplicates and enable ongoing sync

The merge worker solves this by creating a unified profile accessible via both identifiers.

[Erik Andersson]: "It is for the form submits in particular because when you when you submit the form this is created like in your... when we we receive that event from audience that this profile had has a form event submit, we will send that to the CRM system with all of the relevant event data they'll they will then respond saying we created like this specific entity from it... We then do a merge operation."

This same pattern could be applied to the Join CX scenario: after syncing a profile from Join CX (CRM ID keyspace), automatically merge it into the email keyspace to make it discoverable by email address.

---

## Key Technical Details

### Keyspace Mapping Storage

Apsis maintains an integration mapping table structure:

```
installation_id → entity_type → keyspace_discriminator
```

Example for Efficy Enterprise:

```
FC_CORPORATE_INSTALL_001 → person → efficy_enterprise_keyspace_id
FC_CORPORATE_INSTALL_001 → lead → edeal_keyspace_id  (optional entity)
```

This allows the same platform to handle different entity types for the same CRM, directing updates to the correct keyspace automatically.

[Erik Andersson]: "So we create one main key space for the like say contacts person when you install... for each of the entities like the lead entity for FC corporate. Here you see we have the main... We store this mapping for each installation. So like the way we know like if we receive an update for a person, which key space should we update? Well we do and we have that mapping table."

### Profile Update Service Flow

1. Integration receives update from CRM system (containing entity data and CRM ID)
2. Service consults integration mapping table to determine target keyspace
3. Profile is created/updated in the designated keyspace using CRM ID as key
4. (Optional, for Solution 2) Merge worker is called to create secondary access path

---

## Data Consistency Considerations

### Consent Updates: The Constraint

Consent and subscription updates present a unique challenge. Unlike profile updates which include full contact data, consent updates only include:

- `record_id`: The identifier for the consent record
- Consent status/preference data

They do NOT include email addresses or other contact attributes.

[Erik Andersson]: "For consent updates they don't send us any email attributes, they just give us CRM ID... We can't fix this for consent... We will rely on them having that change on their side and if we rely on them having that change on their side, there is no pointing us doing anything manual on our side. They might as well do it for all of the updates."

This is why Solution 3 requires the customer to send email address in the `record_id` field for consent updates—there's no other place the identifier can come from.

---

## Implementation Considerations for Solution 3

### Synchronization Requirement

All API calls from the customer must be consistent:

- Profile creates/updates: email in CRM_ID field ✓
- Consent updates: email in record_id field ✓
- Event submissions: email in profile key field ✓
- Custom integrations reading loyalty points: email in query parameter ✓

[Lukasz Grabowski]: "So we need to synchronize the synchronize the change right? If we say OK, please send us a CRM email address in CRM ID and then let us know when it's done then we will switch the database and that's it."

### Documentation Requirements

[Lukasz Grabowski]: "And then we will document it somewhere that this hack was made for this integration for this customer, you know, because of reasons."

Any permanent solution must be documented with:
- Why this non-standard configuration exists
- Which customer ID this applies to
- What conditions trigger it
- Manual steps required to reapply after reinstallation
- Any customer communication about the workaround nature

---

## Meeting Setup

**Recommended Meeting**: Internal Apsis discussion with product team

**Attendees**:
- Alexander Linzkok (product/technical leadership)
- Opri (product management, if available)
- Erik Andersson (architect, for technical context)
- Lukasz Grabowski (integration owner, for implementation details)

**Customer**: GolfAmore (Intermail/Join CX customer)

**Duration**: 1 hour (45 minutes may be insufficient given complexity)

**Timing**: Day after session (January 22, 2026) at 15:00 (3 PM)

**Purpose**: 
1. Present the problem and three viable solutions
2. Discuss priority vs. other product work
3. Determine if other customers have similar needs
4. Make a recommendation to GolfAmore with supporting rationale

---

## Key Takeaways

1. **The core problem is architectural**: The integration uses CRM ID as the identifier but the customer wants to use email address. These are mutually exclusive without additional translation logic.

2. **Four solutions exist, three are viable**:
   - Solution 1 (full redesign): Too expensive, not recommended
   - Solution 2 (profile merge): Moderate cost, elegant, future-proof, subject to roadmap
   - Solution 3 (database hack): Minimal cost, customer-dependent, requires reinstallation guard, recommended for this case
   - Solution 4 (no change): Maintains original design, unlikely to satisfy customer

3. **Solution 3 is pragmatic for one customer**: Given this is a single, exceptional customer with non-standard use case, the database-level workaround is acceptable if risks are documented and managed.

4. **Consent updates are the constraint**: Because consent updates lack email attributes, any solution requires the customer to send email address in place of CRM ID for **all** update types.

5. **Existing data creates additional complexity**: If customer already has synced profiles, switching keyspaces creates duplicates. This must be addressed separately (merge or delete).

6. **Reinstallation is a breaking change**: If customer reinstalls the integration without documenting the hack, it will revert to CRM ID keyspace and require manual database intervention again.

7. **Documentation is critical**: Whatever solution is chosen must be thoroughly documented given its non-standard nature.

---

## Unresolved Questions & Action Items

### For Product Discussion (Alexander + Opri)
- [ ] Are there other customers requesting similar functionality?
- [ ] Does this warrant adding keyspace selection as a general platform feature?
- [ ] Product priority: where does this rank against other work?
- [ ] If Solution 2 approved: estimated timeline for implementation?

### For Customer (GolfAmore/Intermail)
- [ ] Do they have existing synced profiles? (If yes, they must manually resolve duplicates before switching)
- [ ] Can they commit to sending email address in CRM_ID field for all API calls?
- [ ] Can they send email in record_id field for consent updates?
- [ ] Understanding of reinstallation risk and manual reapplication requirement?

### For Apsis Implementation (if Solution 3 approved)
- [ ] Database update statement with customer ID condition
- [ ] Confluence documentation of the hack and its rationale
- [ ] Runbook for reapplication after customer reinstalls
- [ ] Communication template for customer about non-standard configuration