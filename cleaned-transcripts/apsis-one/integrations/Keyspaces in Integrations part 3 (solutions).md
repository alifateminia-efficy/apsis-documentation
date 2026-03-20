---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One - Integrations
topics: [Keyspace Architecture, Profile Merging, Integration Identifier Mapping, CRM Connector Customization, Email-based Profile Lookup]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Join CX Connector, Keyspace Mapping, Profile Merge Worker Service, Integration Database, CRM ID vs Email Address Identifiers]
session_type: architecture-review
---

## Session Overview

Erik Andersson and Lukasz Grabowski reviewed a complex customer integration problem where an end customer (referred to as Golfamore, managed by Intermail) needs to interact with profiles using email addresses as the primary identifier, while the Join CX loyalty system sync uses CRM IDs. The session examined four possible solution approaches ranging from full system redesign to database-only workarounds, ultimately recommending a lightweight database-level change combined with customer-side API modifications. The team scheduled follow-up discussion with internal product stakeholders (Alexander Lindkvist) to evaluate feasibility and priorities.

---

## Problem Statement: Email-Based Profile Access in CRM Integration

### The Customer Scenario

The Golfamore customer (managed by Intermail) operates a Join CX loyalty system that uses **email addresses as the primary identifier** when they interact with customer profiles. However:

- Join CX syncs customer data to Apsis One using the **CRM ID identifier** (not email)
- The customer has a **direct API connection to Apsis One** where they attempt to query/update profiles
- The customer's API calls use **email addresses as the lookup key**, not CRM IDs
- **The customer has no knowledge of or access to the CRM ID values** in their system

As Erik explained:

> They are integrated with their system and they are only using email addresses when they interact with their system... The issue is that the customer are also directly interacting with Apsis using one API and the customer has no idea what the CRM ID is. Their customers, they only know their email.

### The Fundamental Problem

The customer needs a **translation mechanism**: "I have this email, so which CRM ID is this?" Currently, the architecture provides no way to:

1. Query profiles by email attribute across keyspaces
2. Update a profile created in one keyspace (CRM ID) via a different keyspace (email) 
3. Maintain consistent profiles when the same contact is accessed both ways

**Result of Current State**: Any attempt to interact with profiles via email either:
- Returns **no matching data** (profile doesn't exist in email keyspace)
- Creates **duplicate profiles** (one in CRM keyspace from sync, another in email keyspace from direct API calls)

---

## Solution Option 1: Full Keyspace Configuration (Complex, Not Recommended)

### Approach

Allow customers to specify which keyspace the integration should use during installation instead of bootstrapping a default one.

**Implementation concept**:
- Add dropdown in installation UI: "Select which keyspace to use"
- Pass selected keyspace to generated connector code
- Use conditional logic to route profile operations to the chosen keyspace

### Critical Problems with This Approach

**1. Unique Identifier Validation Issues**

The CRM ID keyspace accepts any string as an identifier, but the email keyspace has strict validation requirements:

> The unique identifier would also need to be configured for it because for the CRM ID for the CRM keyspace like you can specify essentially anything. It can be any string but for the email keyspace, it must be a valid email address.

This means:
- Cannot extract CRM ID as the unique identifier when using email keyspace
- Must extract email addresses instead as the profile key
- Requires profile downloads to extract email addresses rather than contact lookups

**2. Consent/Subscription Data Conflicts**

For consent and subscription updates, the customer would need to send email addresses in the `record_id` field instead of CRM ID—a significant API contract change requiring their system modification.

**3. Single-Email Uniqueness Constraint**

> If there is a key, it's an email. So it's just in one email for one person. It can't be like multiple... it must be unique. So this is impossible, right?

Email keyspace design does not support multiple profiles with the same email address. Customers wanting this functionality **cannot use email keyspace**.

### Effort & Priority Assessment

Erik's assessment:

> This will be a very complicated solution... this will require work from both sides, Apsis and CRM external company side because we need to change the behavior... and also respecting the email address as a key and not CRM ID.

**Estimated effort**: High (weeks) on both Apsis and customer sides. **Not recommended for this single customer.**

---

## Solution Option 2: Automated Profile Merging (Moderate Complexity)

### Approach

When a profile is created/updated via the CRM sync in the Join CX keyspace, **automatically trigger a merge operation** to also create/link the profile in the email keyspace. This preserves the existing CRM-based workflow while making profiles discoverable via email.

**Implementation concept**:
1. Customer syncs contact via CRM (creates profile with CRM ID in Join CX keyspace)
2. Apsis automatically extracts the email from that profile
3. Apsis sends a **merge message** to the Integration Profile Merge Worker service
4. Merge worker creates/links the same profile in the email keyspace using the email as the key
5. Profile is now accessible via both CRM ID (in Join CX keyspace) and email (in email keyspace)

### How Profile Merge Worker Currently Works

The merge service is primarily used for **form submission handling**:

> When you submit the form... when we receive that event from Audience that this profile had has a form event submit, we will send that to the CRM system with all of the relevant event data... we receive a ID back. We then do a merge operation. So we merge this profile that exists right now, like in your form or email keyspace. We merge that with our silhouette keyspace...

Merge operations handle scenarios like:
- Form submissions creating leads that later become contacts (need to merge lead and contact profiles)
- CRM detecting duplicate contacts that should be unified
- Cross-keyspace profile consolidation

### Advantages

- **Reuses existing infrastructure**: The merge worker and merge logic already exist
- **No API changes needed** from customer (they keep sending CRM IDs)
- **Maintains backward compatibility** with existing integrations
- **Realistic implementation**: 2-3 days estimated effort on Apsis side

### Implementation Details

The merge would be triggered in the **profile update service**:

1. After normal profile creation/update (with CRM ID, in Join CX keyspace)
2. Extract email attribute from the updated profile
3. Create merge message with:
   - Source keyspace: Join CX keyspace
   - Source key: CRM ID
   - Destination keyspace: Email keyspace
   - Destination key: Extracted email address
4. Send to merge worker for profile consolidation

**For this single customer**, the destination keyspace ID could be **hard-coded** in the profile update service (not elegant but acceptable for one-off implementation).

### Limitations

- Only merges profiles that have email addresses; profiles without email remain unmapped
- Duplicate profiles may exist if customer has old profiles in both keyspaces (customer owns cleanup)
- If customer reinstalls the integration, the setup must be redone

### Effort Assessment

> This would be quite straightforward for you... it would take you like 2-3 days. Maybe it wouldn't be that much... you can do this as a specific implementation for this customer.

---

## Solution Option 3: Database-Level Keyspace Remapping (Lightweight, Recommended)

### Approach

**Change only the integration database configuration** to point to the email keyspace instead of the Join CX keyspace. No code or logic changes needed.

### How Integration Keyspace Mapping Works

Apsis stores a **mapping table** in the integration database that tracks which keyspace to use for each entity type per installation:

```
Installation: Golfamore (Join CX instance)
Entity Type: member
Mapped Keyspace: [join_cx_keyspace_id]  ← Can be changed to email_keyspace_id
```

When an integration receives an update for a `member` entity, it looks up this mapping table to determine which keyspace to write to.

> We store this mapping for each installation. So like if we receive an update for a person, which key space should we update? Well we have that mapping table. So for this FC corporate instance, if we receive a update for a person, then we will do all the updates in this like we will utilize this key space discriminator for that.

### Implementation

1. **Pre-requisite communication**: Instruct customer to send **email addresses in the `CRM_ID` field** instead of actual CRM IDs for all API calls (including consent updates)
2. **Database change**: Update the keyspace mapping in the integration database table:
   ```
   UPDATE integration_mappings 
   SET keyspace_id = 'email_keyspace_id' 
   WHERE installation_id = 'golfamore_id' 
   AND entity_type = 'member'
   ```
3. **Verification**: Integration now routes all member updates to email keyspace

### Why This Works Despite the Caveat

The critical constraint is that **email keyspace requires valid email addresses as the profile key**, not arbitrary strings. This solution works **only if the customer sends email addresses in the CRM_ID field**:

> We will still look for the CRM ID as the identifier, but as we mentioned in the beginning, this will not work if we try to update the email keyspace. We would still need... the email addresses provided to us as the CRM ID.

When customer provides email in the `CRM_ID` field:
- Apsis receives update for member with `CRM_ID = "customer@example.com"`
- Looks up keyspace mapping → routes to email keyspace
- Uses `customer@example.com` as the profile key in email keyspace (valid email ✓)
- Profile updates work correctly

### Critical Caveats and Risks

**1. Consent/Subscription Record ID Changes**

Consent and subscription updates require the `record_id` field. Currently, customers send:
```
record_id: [CRM_ID]  // e.g., "12345"
```

With email keyspace, they must send:
```
record_id: [email_address]  // e.g., "customer@example.com"
```

This is a customer-side API contract change they must implement.

> For consent updates they don't send us any email attributes, they just give us CRM ID. So so we can't fix this for consent map it like we will rely on them having that change on their side.

**2. Reinstallation Risk**

When an integration is reinstalled, the database mapping reverts to the default keyspace (Join CX). The customer or Apsis must remember to re-apply this database change.

> The caveat here is of course if they for whatever reason were to reinstall the installation, then this change must again be done on the side because we will have reverted to the normal joint CX in the database because that's that's set up on installation.

**3. Existing Profile Duplicates**

If the customer already has existing profiles synced via Join CX (in the CRM keyspace), changing the keyspace mapping does **not automatically migrate** them. The customer will face:
- Old profiles in Join CX keyspace (from prior syncs)
- New profiles in email keyspace (from future syncs)
- Potential duplicate contact representations

> If they already have existing contacts, they would exist of course in the join CX key space. If we change which key space, everything would be duplicated unless they do manual merge work before or they just delete every profile they have.

**Solutions for existing profiles**:
- Customer deletes all existing profiles before the keyspace change (acceptable if this is early in integration)
- Customer performs manual merge work to consolidate old and new profiles
- Customer runs a full resync after keyspace change (if Apsis supports this)

Erik noted uncertainty about whether this customer has existing profiles:

> If we checks if there are existing customers, I might be speaking, I might be telling you lies though it for this customer that maybe they actually aren't already connected, but I know they have other customers at least, maybe not this one. This could be the blocker for them to proceed.

### Advantages of This Approach

- **Minimal Apsis-side effort**: Can be implemented in 5 seconds (one database record change)
- **No code changes**: No risk of introducing bugs in logic
- **Fast customer enablement**: Apsis can implement same day if customer is ready
- **No redesign of integration architecture**: Respects the existing design pattern
- **Customer controls timeline**: Unlike solution 2 (waiting on Apsis priorities), customer can implement their side immediately and request the database change whenever ready

Erik's strong endorsement:

> This is a quite quick... I really like it. I don't have any issues with this... As I said, having in mind this is only for one customer, uh, I would say this hack is OK... Me personally, I am a big proponent of this. If anything had to be done on Apsis side, I would go for the merge one.

### Documentation & Change Control

Given this is a one-off customization, it must be:

> Document it somewhere that this hack was made for this integration for this customer, you know, because of reasons.

Recommended documentation location: Confluence page for Golfamore/Intermail integration with:
- Rationale for non-standard keyspace selection
- Date of change
- Customer notification of reinstallation risks
- Contact for reverting if needed

---

## Solution Option 4: No Changes (Customer Follows Designed Pattern)

### Approach

Advise customer to use the integration as designed:
- Use **CRM IDs** as the primary identifier
- Maintain the Join CX keyspace relationship
- Update their API calls to lookup/send CRM IDs instead of emails

### Rationale

The Join CX connector was intentionally designed around CRM IDs. The email-based access pattern is not the intended use case.

> The system this integration was designed to rely on CRM ID and not email, right?

### Why Customer Likely Won't Accept This

The customer's business process is fundamentally email-centric. Requiring them to reverse-map every API request through CRM ID lookup is operationally impractical and may not be technically feasible in their system.

This option is mentioned for completeness but flagged as unlikely to satisfy the customer.

---

## Comparison of Solutions

| Aspect | Option 1: Config | Option 2: Merge | Option 3: DB Remap | Option 4: No Change |
|--------|------------------|-----------------|-------------------|-----------------|
| **Apsis Implementation** | Complex | 2-3 days | 5 seconds | None |
| **Customer Implementation** | None | None | Immediate API change | Full redesign |
| **Code Changes** | Yes (connector logic) | Yes (merge trigger) | No | No |
| **Risk to Other Integrations** | High | Low | None | None |
| **Time to Enable Customer** | Weeks | 2-3 days | Hours | N/A |
| **Architectural Respect** | Breaks design | Preserves | Preserves | Preserves |
| **Scalability** | Yes (future customers) | Yes (existing capability) | No (one-off) | N/A |
| **Recommendation** | ❌ Not viable | ✅ Best if Apsis prioritizes | ✅ Recommended short-term | ❌ Unlikely to accept |

---

## Recommended Path Forward

**Primary recommendation: Solution Option 3 (Database Remapping)**

Reasons:
- Fastest path to customer enablement (same day possible)
- Minimal risk to Apsis system (no code changes)
- Minimal customer effort (focused API change)
- Acceptable for single-customer use case

**Secondary recommendation: Solution Option 2 (Profile Merge)**

If Apsis decides this customer request signals broader demand:
- Enables future customers to use email keyspace without their API changes
- 2-3 day investment reusable for multiple customers
- More architecturally sound long-term

**Next steps**:
1. Confirm whether customer already has existing synced profiles (determines viability of Option 3)
2. Schedule meeting with Alexander Lindkvist and possibly Opri to discuss priorities
3. Present Options 2 and 3 to customer with risks/requirements clearly outlined
4. Let customer and product team jointly decide based on business value vs. effort

---

## Key Implementation Questions and Answers

### "What if the profile doesn't exist in the email keyspace when we merge?"

> If there it doesn't exist, I think they create an empty profile. So that's that's not that's not a problem... we just put the key space ID and email key space ID and of course source... and the email extracted email attribute from this CRM entity.

The merge operation can create a new profile in the destination keyspace if it doesn't already exist.

### "What happens with old profiles if we change the keyspace mapping?"

The database change only affects **future** updates. Existing profiles in the old keyspace remain unchanged unless:
- Customer manually merges them
- Customer deletes them before the change
- Customer re-syncs all profiles after the keyspace change

This must be clarified with the customer before implementing Option 3.

### "Will consent updates work with Option 3?"

No, unless the customer changes their API calls. The consent update endpoint expects:
```
{
  "record_id": "value_in_mapped_keyspace",
  ...
}
```

Currently they send CRM IDs. With email keyspace, they must send email addresses.

### "Can this break if the customer reinstalls?"

Yes. The database mapping is set during installation. A reinstall reverts to default keyspace. Mitigation:
- Document the change with customer
- Include in integration runbook for future administrators
- Apsis can flag this customer as needing the mapping re-applied post-reinstall

---

## Risk Assessment and Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Existing profiles create duplicates (Option 3) | Medium | High | Verify with customer; document cleanup steps |
| Consent updates fail due to ID mismatch | High | High | Clearly communicate API contract changes before implementation |
| Reinstallation reverts configuration | Medium | Medium | Document prominently; add to integration checklist |
| Similar requests from other customers | Medium | Medium | Monitor; use as input for long-term architecture decision (Option 2) |
| Customer provides non-email values in CRM_ID field | Low (with clear communication) | Critical | Provide validation/testing before production change |

---

## Follow-Up Meeting Scheduled

**Meeting Title**: Golfamore / Intermail Integration Solution Discussion

**Participants**:
- Erik Andersson (Apsis architecture/integration)
- Lukasz Grabowski (Integration development lead)
- Alexander Lindkvist (Internal Apsis stakeholder)
- Opri (optional, at Alexander's discretion)

**Timing**: Tomorrow (January 22, 2026) at 3 PM (15:00)

**Duration**: 1 hour

**Agenda**:
1. Present three viable solutions (Options 2, 3, 4)
2. Assess whether similar customer requests exist
3. Determine Apsis priority for Options 2 vs. 3
4. Identify any architectural considerations from product perspective
5. Plan next communication with customer

**Shared Documentation**:
- Erik to share detailed solution diagrams and notes with Lukasz before meeting
- Confluence page to be created for Golfamore integration with full technical context

---

## Key Takeaways

1. **The core problem is solvable** but represents a deviation from the designed integration pattern (CRM IDs → Apsis). Solutions exist at different cost/complexity levels.

2. **Email keyspace has strict requirements** for profile keys (must be valid email addresses). This constrains solutions to either:
   - Having customer provide emails as the identifier
   - Building special merge logic to maintain dual-keyspace mapping

3. **Solution Option 3 (Database Remapping) is the pragmatic recommendation** for this single customer:
   - Apsis changes one database record (5-second effort)
   - Customer changes API calls to use emails instead of CRM IDs (moderate effort, under their control)
   - Customer can implement immediately without waiting on Apsis dev priorities
   - Acceptable one-off customization for non-standard use case
   - Clear documentation of the deviation is essential

4. **Solution Option 2 (Automated Merging) is the architecturally sound choice** if:
   - This customer request indicates broader market demand
   - Apsis prioritizes this work (estimated 2-3 days)
   - Long-term vision includes email-based access to CRM-synced profiles
   - Can be reused for future customers with similar requirements

5. **Existing profile duplicates are a blocker** for Option 3 if customer already has synced data. Must verify this before proceeding.

6. **Reinstallation is a documented risk** for Option 3. Customer must be aware that re-installing the integration reverts the keyspace change and requires manual re-application.

7. **This is fundamentally a business prioritization decision**, not a technical impossibility. Erik's philosophy applies:

> That's the beauty of developing. We can do anything. The question is always effort, priorities and philosophies.

---

## Unresolved Questions for Follow-Up

- [ ] Does the Golfamore customer already have existing profiles synced in the Join CX keyspace? (Determines feasibility of Option 3)
- [ ] Are there other Intermail customers with similar email-based access requirements? (Informs whether to invest in Option 2)
- [ ] What is the product roadmap for keyspace flexibility? (Long-term direction)
- [ ] Can Apsis provide email keyspace ID in documentation so customer knows the value for their API changes?
- [ ] Should validation be added to prevent non-email values being sent in CRM_ID field when using email keyspace?