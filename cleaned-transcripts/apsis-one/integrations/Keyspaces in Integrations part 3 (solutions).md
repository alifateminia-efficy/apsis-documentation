---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One Integrations
topics: [Keyspace configuration, Profile identifier mapping, Multi-keyspace merge strategies, Generic connector customization, CRM integration workarounds]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Apsis One, Join CX (Intermail), Generic Connector, Keyspace system, Profile merge service, CRM ID identifiers, Email keyspace]
session_type: architecture-review
subdomains: ["Different Types of Connectors"]
---

## Session Overview

This session documents a technical discussion between Erik Andersson and Lukasz Grabowski regarding a complex integration scenario with a Join CX (Intermail) customer who wants to use email addresses as profile identifiers instead of the standard CRM ID. The customer uses Join CX for loyalty management and syncs data to Apsis One via the generic connector, but their direct API interactions use email addresses as the unique identifier. The session explores four potential solutions, ranging from full-fledged architectural changes to pragmatic database-level workarounds, ultimately recommending a lightweight approach that requires minimal changes on Apsis One's side while placing some burden on the customer's integration work.

---

## Problem Statement: Keyspace Identifier Mismatch

### The Customer's Technical Setup

The end customer operates through Intermail's Join CX loyalty system and integrates with Apsis One. However, they have two distinct interaction patterns:

1. **Via Join CX (the CRM partner):** Uses CRM ID as the unique identifier. When Join CX syncs contacts to Apsis One, they are registered in the **Join CX keyspace** using CRM ID as the profile key.

2. **Via direct API:** The customer's own API interactions use **email addresses as identifiers**, not CRM IDs. They have no knowledge of or capability to reference CRM IDs.

This creates a fundamental mismatch: the same person can exist as two separate profiles in Apsis One—one keyed by CRM ID in the Join CX keyspace, and another keyed by email in the email keyspace.

### The Core Problem

[Erik Andersson]: "The unique identifier would also need to be configured for it because for the CRM ID for the CRM key space like you can specify essentially anything. It can be any string but for the e-mail key space it must be a valid e-mail address."

When the customer tries to query or update profiles via their API using email addresses:
- Profile lookups fail (no matching profile found in CRM ID keyspace)
- Direct profile creation/updates generate duplicate profiles in the email keyspace instead of updating the existing profile synced from Join CX
- Any custom event extraction or data retrieval returns no results

[Lukasz Grabowski]: "So like from the high level, it looks like there is no mapping."

[Erik Andersson]: "Whatever their use case, it will either create duplicates or return no matching data."

---

## Solution Option 1: Full Keyspace Configuration (High Complexity)

### Approach

Allow customers to specify at installation time which keyspace the integration should use. Instead of the system bootstrapping a default keyspace for all Join CX installations, this installation would be configured to use the email keyspace.

### Implementation Requirements

1. Add a dropdown selector in the installation UI: "Select which keyspace to use"
2. Pass the selected keyspace choice to the connector code generation
3. Update the integration logic to respect this configuration

### Complications

**Unique Identifier Validation:** The system cannot treat email addresses identically to CRM IDs. Email keyspace identifiers must be valid email addresses, not arbitrary strings. This requires:
- Extracting the email attribute instead of the CRM ID from source contacts
- Validating that email addresses are present and valid
- Different code paths for profile download and identifier extraction

**Consent/Subscription Handling:** When processing consent or subscription updates, the customer would need to send the email address in the record ID field rather than the CRM ID—a significant change to their API contract.

[Erik Andersson]: "For consent subscriptions they would need to say the record ID is not the CRM ID, the record would be the e-mail address instead because otherwise it will be problematic."

### Mutual Exclusivity Constraint

[Lukasz Grabowski]: "If there is a key, it's an e-mail. So it's just in one e-mail for one person. It's cannot be like multiple."

Supporting multiple profiles with the same email address becomes impossible. The customer must accept that email is a unique identifier with a one-to-one mapping to profiles.

### Assessment

This solution requires substantial effort on both Apsis One and the customer's side. It would typically be a 3+ month priority item and involves redesigning core connector behavior.

[Erik Andersson]: "I am not really suggesting that we go with this approach. I'm just lifting it to show you what's like there the different objects there are."

---

## Solution Option 2: Automatic Merge After Profile Creation (Medium Complexity)

### Approach

When a profile is created or updated via the Join CX keyspace, automatically trigger a merge operation with the email keyspace. The system would:
1. Create/update the profile in the Join CX keyspace (using CRM ID) as normal
2. Extract the email attribute from the profile
3. Send a merge request to combine the Join CX keyspace profile with any profile in the email keyspace

### How Profile Merge Works Today

[Erik Andersson]: "For the form submits in particular because when you when you submit the form this is created like in your form key space or the e-mail and SMS key space when it is submitted when we receive that event from audience that this profile had has a form event submit, we will send that to the CRM system with all of the relevant event data they'll they will then respond saying we created like this specific entity from it."

The merge worker service already exists and is actively used for form submission handling. When a lead is created in the CRM:
1. Apsis One receives the new CRM ID
2. A merge operation combines the form-submission profile (in form/email keyspace) with the new CRM profile (in CRM keyspace)
3. This prevents duplicates and ensures one canonical profile

[Erik Andersson]: "We have a merge worker in integration already like it takes merge. We take a source key and the source key space and a the other is a destination. The key and the destination key space and you would always have the source key and the source key space because you have that in the sync already with the CRM ID and the join CX one."

### Implementation Details

**Estimated Effort:** 2-3 days

**Code Pattern:**
```
After profile creation/update in Join CX keyspace:
IF profile has email attribute THEN
  Create message for integration_profile_merge_worker_service
  With parameters:
    - source_keyspace_id: <join_cx_keyspace_id>
    - source_profile_key: <crm_id>
    - destination_keyspace_id: <email_keyspace_id>
    - destination_profile_key: <email_address>
```

For an initial implementation targeting a single customer, the keyspace IDs could be hardcoded; a future enhancement could expose this as configurable in the UI.

### Advantages

- Uses existing, proven infrastructure (merge service)
- No logic redesign required
- All work stays within Apsis One's codebase
- Solves duplicate problem automatically
- Work is prioritizable against internal roadmap

### Disadvantages

- Subject to Apsis One's development priorities (could be delayed 3-4 months)
- Customer must wait for implementation

---

## Solution Option 3: Database-Level Keyspace Mapping Change (Minimal Complexity, Recommended)

### Approach

The system maintains an internal mapping table that associates each installation with a keyspace discriminator. For example:

```
integration_keyspace_mappings
├─ fc_corporate_instance → fc_corporate_keyspace (main keyspace)
├─ join_cx_instance → join_cx_keyspace (main keyspace)
└─ intermail_customer → email_keyspace (overridden)
```

[Erik Andersson]: "For each integration like that we create a key space for each of the entities. So we create one main key space for the like say contacts person when you install. So this for example if you have any optional entities like the lead entity for FC corporate."

When a profile update arrives for a given integration, the system queries this mapping to determine which keyspace to update.

**The Workaround:** Simply change the mapping entry in the database from `join_cx_keyspace` to `email_keyspace`. No code changes required on Apsis One's side.

[Erik Andersson]: "We would just go into the database and change this to the e-mail key space then the integration. Whenever there is an update for member, it will try and do those updates on the e-mail key space instead and this is without changing any code or logic."

### The Critical Caveat

The system will still attempt to use the CRM ID as the profile identifier. For email keyspace operations, this fails—email keyspace requires valid email addresses as keys.

**Solution:** The customer must send email addresses in the CRM ID field of all requests.

[Erik Andersson]: "We will still look for the CRM ID as the identifier, but as we mentioned in the beginning, this will not work if we try to update the e-mail key space. We would still need the e-mail addresses provided to us as the CRM ID."

### Required Customer Changes

1. **Profile Updates:** Send email address in the CRM ID field instead of the actual CRM ID
2. **Consent/Subscription Updates:** Send email address in the record ID field
3. **All Integration Touchpoints:** Ensure email addresses are provided wherever CRM ID was previously used

[Erik Andersson]: "Also when it comes to the consent updates, they can't send us. They can send us like the CRM ID on the record ID as they normally do. They have to send us the e-mail address in the record ID."

### Timeline and Dependencies

**Apsis One Work:** ~5 minutes (one database update)  
**Customer Work:** 1 week implementation (they can proceed independently)

Because the customer's work is decoupled from Apsis One's development priorities, they can implement changes immediately without waiting for a release cycle.

[Erik Andersson]: "They can do this like in the next week. That is not. They don't need to sit and wait for you to finish your business priorities, which might be finished like in three months time."

### Risk: Reinstallation

If the customer reinstalls the integration (deletes and recreates it), the database mapping reverts to the default `join_cx_keyspace`. The override must be reapplied.

[Erik Andersson]: "The caveat here is of course if they for whatever reason were to reinstall the installation, then this change must again be done on the side because we will have reverted to the normal joint CX in the database because that's that's set up on installation."

This is unlikely in production but worth documenting.

### Risk: Existing Profile Duplication

Profiles already synced from Join CX exist in the `join_cx_keyspace`. When the mapping changes to `email_keyspace`, future updates will operate on the email keyspace, creating a duplicate set of profiles.

[Erik Andersson]: "If they already have existing contacts, they would exist of course in the join CX key space. If we change which key space, everything would be duplicated unless they do manual merge work before or they just delete every profile they have that is of course, also an option."

**Resolution Options:**
1. Full resync: Trigger a full data synchronization after the keyspace change. All contacts download again with email identifiers, creating new profiles in the email keyspace. The merge service can then consolidate duplicates.
2. Manual cleanup: Customer deletes all existing profiles in Apsis One and starts fresh with the email keyspace
3. Accept duplication: Keep old profiles in join_cx_keyspace; new syncs go to email_keyspace (not recommended)

### Advantages

- **Minimal Apsis One effort:** ~5 minutes of database work
- **Fast customer deployment:** They can implement within days
- **No architectural changes:** Works within existing system design
- **No blocking priorities:** Not dependent on product roadmap
- **Proven infrastructure:** Uses existing keyspace and mapping mechanisms

### Disadvantages

- **Workaround, not a proper solution:** Relies on non-standard identifier usage (email in CRM ID field)
- **Requires customer compliance:** They must implement changes on their side
- **Risk on reinstallation:** Database change must be reapplied after reinstalls
- **Not scalable:** Better suited as a one-off workaround than a general feature

### Assessment

[Lukasz Grabowski]: "I I really like it. I don't have any issues with this, but I'm wondering if you can see any other issues we will have later."

[Erik Andersson]: "Not if we change anything, as long as they replace it on every, as long as they give CRM ID in every place that is not on."

---

## Solution Option 4: Customer Implements the Mapping (No Apsis Changes)

### Approach

The system operates as designed. The customer accepts the reality that:
- Apsis One uses CRM IDs and Join CX keyspace
- They must maintain their own mapping between email addresses and CRM IDs in their integration layer

Their custom integration code would perform the lookup: "When I receive an email address, which CRM ID does this correspond to?" Then use the CRM ID for all Apsis One interactions.

### Assessment

[Erik Andersson]: "I don't think this is going to be an option, but of course the 4th solution here is that APSIS does nothing and the customer utilizes the CRM ID and the join CX key space in their custom solution is that meaning they would need to essentially actually do the work between the join CX instance instead."

While technically sound and aligned with the system's designed behavior, this shifts the entire burden to the customer and defeats the purpose of having the generic connector integration.

[Lukasz Grabowski]: "The system this integration one was designed to rely on CRM ID and not e-mail, right?"

---

## Recommended Solution: Option 3 with Option 2 as Future Enhancement

### Near-Term: Option 3 (Pragmatic Workaround)

For this specific customer and use case:
1. Change the keyspace mapping in the database to `email_keyspace`
2. Customer sends email addresses in the CRM ID field across all API touchpoints
3. Document the change in an internal knowledge base with reasons and risks
4. Customer performs a full resync to establish email keyspace profiles
5. Document the reinstallation risk

### Medium-Term: Option 2 (Proper Solution)

Prioritize the merge-based solution as a proper feature if:
- Additional customers request similar functionality
- Product strategy indicates multi-keyspace support is valuable
- The workaround proves problematic in production

---

## Implementation Coordination

### Stakeholders to Involve

**Internal (Apsis One):**
- Erik Andersson (Architecture)
- Alexander Linzkok (Product/Engineering Lead)
- Opri (Product, optional depending on Alexander's assessment)

**Customer:**
- Intermail representative

### Discussion Topics for Internal Meeting

1. **Preferred solution:** Option 3 (immediate) vs. Option 2 (proper but delayed)
2. **Risk assessment:** Reinstallation scenarios, existing profile handling
3. **Communication plan:** What to tell the customer about workaround nature
4. **Documentation requirements:** Where and how to record this exception
5. **Future direction:** Whether to build Option 2 as a general feature

---

## Key Takeaways

1. **Root Cause:** Identifier mismatch between two integration patterns—Join CX uses CRM IDs, direct API uses emails. The generic connector was designed assuming a single identifier throughout.

2. **No Perfect Solution Exists:** Each option involves tradeoffs between effort, time-to-market, and design integrity.

3. **Option 3 is Pragmatic for One Customer:** A database-level mapping change with customer-side email-in-CRM-ID workaround is fastest and unblocks the customer without substantial Apsis One development.

4. **Option 2 is the Proper Architecture:** Using the existing merge service to automatically align profiles across keyspaces is how this should work long-term, but it requires development prioritization.

5. **Documentation is Critical:** Any workaround must be clearly documented with implementation details, risks (especially reinstallation), and rationale.

6. **Customer Coordination is Essential:** Success depends on customer willingness to send email addresses as CRM IDs and execute a full data resync to handle existing profile duplication.

---

## Unresolved Questions & Action Items

1. **Do existing profiles need merging?** Confirm whether the customer already has synced profiles in the join_cx_keyspace that must be consolidated with email_keyspace profiles post-implementation.

2. **Is reinstallation a real risk?** Clarify customer's operational practices—do they reinstall integrations? If yes, document procedures to reapply the database change.

3. **Customer API capability:** Verify the customer can modify their API to send email addresses in the CRM ID field without breaking other integrations or downstream systems.

4. **Confluence documentation:** Create a confluence page documenting this workaround, including:
   - Why it was done (email keyspace preference vs. CRM ID design)
   - How it works (database mapping override + email-as-identifier)
   - Implementation steps
   - Risks and mitigations
   - Reinstallation procedures

5. **Scheduled Follow-Up Meeting:**
   - **Date/Time:** Tomorrow at 3:00 PM
   - **Attendees:** Lukasz Grabowski, Erik Andersson, Alexander Linzkok, (Opri optional)
   - **Topic:** "Intermail/Golf Amore Customer—Keyspace Configuration Solutions"
   - **Duration:** 1 hour
   - **Goal:** Align on recommended solution and next steps with customer