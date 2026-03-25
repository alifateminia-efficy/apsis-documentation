---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One Integrations
topics: [Keyspace Configuration, Profile Merging, Integration Architecture, Customer Use Cases, CRM ID vs Email Identifier, Generic Connector Behavior]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Apsis One, Join CX (Intermail), Generic Connector, Keyspaces, Profile Merge Worker, Integration Database, CRM ID, Email Keyspace]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow]
---

## Session Overview

This session discusses a complex customer scenario where an end customer (using Intermail's Join CX platform) wants to manage profiles in Apsis One using email addresses as the primary identifier, rather than CRM IDs. The team explores the architectural constraints and evaluates four potential solutions ranging from full platform redesign to database-level workarounds. The discussion preserves critical knowledge about keyspace mechanics, merge operations, and the practical trade-offs between implementation effort and solution elegance.

---

## The Problem: Email as Primary Identifier vs. CRM ID Design

### Customer Scenario

The end customer uses **Join CX**, a loyalty system developed by **Intermail**. They interact with Join CX using **email addresses as their only identifier**. This data syncs to Apsis One through the generic connector.

However, the customer also directly integrates with Apsis One's API independently, and they:
- Only know email addresses
- Have no knowledge of CRM IDs
- Want to use email addresses to look up and update profiles in Apsis

### Why This Breaks Today

[Erik Andersson]: The generic connector is designed to use **CRM ID as the unique identifier** within the **Join CX keyspace**. When the customer tries to interact via email:

- **Scenario 1 - Read operations fail**: If they query for a profile by email, they find nothing because the profile exists only under the CRM ID key in the Join CX keyspace.
- **Scenario 2 - Write operations create duplicates**: If they try to update a profile by email, a new profile is created in the **email keyspace** instead of updating the existing one in the Join CX keyspace. This creates duplicate profiles—one keyed by CRM ID, one by email.

[Lukasz Grabowski]: The core issue is that they want to treat **email as a profile key** (the unique identifier for that keyspace), but the system is designed with CRM ID as the key.

### Architectural Constraint: Keyspaces Require Unique Keys

[Lukasz Grabowski]: If email is a keyspace key, it must be unique. A person cannot have multiple profiles under the same email address—that would violate the keyspace contract.

[Erik Andersson]: Exactly. This is a prerequisite that cannot be negotiated. If they want to use the email keyspace with email as the key, they cannot have multiple profiles per email.

---

## Solution 1: Configurable Keyspace at Installation (Complex)

### Approach

Allow customers to specify which keyspace the integration should use during installation, rather than having it hard-coded to the Join CX keyspace.

### What Changes

- Add a dropdown in the installation UI: "Select which keyspace to use"
- For this customer's installation, select the email keyspace
- The generated connector code would route all operations to that keyspace

### Why This Doesn't Work Alone

[Erik Andersson]: Even if you specify the keyspace, the **unique identifier extraction logic breaks**:

For the **CRM ID keyspace**, you can extract any string value as the identifier (contact ID, etc.). For the **email keyspace**, the identifier must be a valid email address—you cannot use CRM ID as the key.

This means:
- Code that extracts CRM IDs and uses them as profile keys would fail
- You'd need to extract email addresses instead
- The extraction logic varies per keyspace

Additionally, for **consent/subscription updates**, the customer would need to send email addresses in the `record_id` field instead of CRM IDs. This requires changes on their API integration side.

### Effort Assessment

[Erik Andersson]: This is highly complex because:
1. The connector code generation would need conditional logic for each keyspace
2. The unique identifier extraction would need to be keyspace-aware
3. Both Apsis and the customer's system need changes
4. It's unclear if this is a pattern worth building for reuse

**Not recommended as primary solution.**

---

## Solution 2: Automatic Profile Merging After Sync (Moderate Effort)

### Approach

After the generic connector syncs a profile to Apsis (using CRM ID in the Join CX keyspace), automatically initiate a **merge operation** to link it to the email keyspace.

### How It Works

1. Customer sends an update for a contact (including email address)
2. Apsis creates/updates the profile in the Join CX keyspace as usual (keyed by CRM ID)
3. A message is sent to the **integration profile merge worker service** with:
   - **Source**: Join CX keyspace + CRM ID
   - **Destination**: Email keyspace + email address (extracted from the profile)
4. The merge worker creates a link between the two keyspace entries, preventing duplicates

### Profile Merge Worker Context

[Erik Andersson]: The profile merge worker already exists and is primarily used for **form submission handling**:

> When a form is submitted, a profile is created in the form keyspace (or email/SMS keyspace). The CRM then creates an entity (e.g., a lead in E-deal) and returns an ID. We merge the form profile with the CRM keyspace profile so they reference the same contact and future updates go to the right place.

For this customer, we'd apply the same pattern in reverse—merge from CRM keyspace to email keyspace.

### Implementation

[Lukasz Grabowski]: Can you write 3-5 days for the estimate?

[Erik Andersson]: Yes. We already have the merge worker libraries. The implementation is straightforward:
- In the profile update service, after creating/updating the profile
- Check if this is the target customer
- Extract the email attribute
- Send a merge message to the worker with the keyspace IDs and email

If done as a customer-specific implementation (hard-coded), this is **2-3 days of work**.

### What Still Requires Customer Changes

For **consent updates**, the customer doesn't send email attributes—they only send CRM ID. The merge step only works if there's an email address to use as the destination key. So:
- The customer must still send email addresses in relevant API fields
- This is less burdensome than Solution 1 but still requires coordination

### Trade-offs

**Pros**:
- Proper long-term solution
- Reusable if other customers request it
- Merge worker is proven technology
- Leaves data model unchanged

**Cons**:
- Requires Apsis development time
- Subject to product priorities (Erik estimates 3-4 months in a typical backlog)
- Customer work is still required for consent updates

---

## Solution 3: Database Mapping Workaround (Minimal Effort, Risk-Managed)

### Approach

The integration system maintains a **mapping table** that tracks which keyspace should be used for each entity per installation:

```
For installation ID X:
  Entity "member" → Use keyspace "join_cx"
  
For installation ID Y:
  Entity "member" → Use keyspace "email"
```

Instead of changing code or logic, simply **change the mapping in the database** for this one customer.

[Erik Andersson]: Instead of:
```
Installation → member → join_cx_keyspace_id
```

Change it to:
```
Installation → member → email_keyspace_id
```

Now whenever the connector processes a member update, it routes to the email keyspace automatically.

### Effort

[Erik Andersson]: Five seconds. No code changes. No deployment.

### The Catch: CRM ID Must Become Email Address

The system still looks for **CRM ID as the identifier**, but the email keyspace expects email addresses as keys.

**Solution**: Ask the customer to send email addresses in the `CRM_ID` field.

When they call the API:
```
POST /api/contacts
{
  "crmId": "customer@example.com",  // MUST be email, not their internal CRM ID
  "name": "John Doe",
  "email": "customer@example.com"
}
```

If they do this consistently, everything works:
- Profile gets created/updated in email keyspace under the email key
- Customer can query by email and find profiles
- No duplicates because email is the keyspace key

### Risk 1: Existing Profiles Become Orphaned

[Lukasz Grabowski]: What about old profiles that exist in the Join CX keyspace (keyed by their old CRM IDs)?

[Erik Andersson]: They stay in the Join CX keyspace. The customer will have:
- Old profiles: in `join_cx` keyspace, keyed by CRM ID
- New profiles: in `email` keyspace, keyed by email

Unless they:
1. Do a **full sync** where every contact is re-downloaded and updated with email in the CRM ID field (triggering the merge)
2. Or manually merge profiles themselves
3. Or delete all old profiles and start fresh

This is something the customer must handle.

### Risk 2: Reinstallation Breaks the Workaround

[Erik Andersson]: If the customer ever reinstalls the integration, the database mapping reverts to the default (join_cx keyspace). The workaround would need to be re-applied.

This is a maintenance risk that must be documented.

### Risk 3: Consent Updates Still Require Changes

For consent/subscription updates, they must send email addresses in the `record_id` field, not CRM IDs. This is a system-wide requirement, not a workaround.

### Why Erik Favors This Solution

[Erik Andersson]:
> This is the swift solution. I can do this for them this afternoon. The advantage is they don't need to wait for product priorities—they can implement their side in a week if they want.

The customer controls the timeline for their API changes, while Solution 2 would put them in a queue for Apsis development.

### Documentation Requirements

[Lukasz Grabowski]: We need to document:
- Why this workaround was implemented (exceptional case, one customer)
- That the customer must always send email in CRM ID field
- Reinstallation risk
- Orphaned profile scenario

---

## Solution 4: Do Nothing — Customer Adopts CRM ID Approach (Simplest)

### The Ask

The customer stops trying to use email as a key and instead:
- Uses CRM IDs in their custom integration
- Maintains a mapping between email and CRM ID on their side
- Calls Apsis APIs with CRM IDs

### Why It's Valid

[Lukasz Grabowski]: This is how the system was designed. The generic connector is built around CRM IDs for a reason—it's an identifiable, stable identifier that doesn't have uniqueness or format constraints.

[Erik Andersson]: Exactly. The customer can still achieve their goals; they just do the translation work between email and CRM ID themselves.

### Why They Probably Won't Accept It

The customer specifically wants Apsis to handle email-based lookups. They're asking for a change to accommodate their use case, not to accept the system as-designed.

### Value of Mentioning It

[Lukasz Grabowski]: We should still present this option to them. It clarifies that the system has a design philosophy, and we're being asked to deviate from it.

---

## Recommended Path Forward

### Preferred Solution: Solution 3 (Database Workaround)

[Erik Andersson]:
> If anything has to be done, I would go for the merge one. But if we're looking at the quickest path, this database mapping change is the pragmatic choice.

[Lukasz Grabowski]:
> Having in mind this is only for one customer and this is an exceptional case, I really like this. I don't have any issues with this.

### Rationale

1. **Minimal Apsis effort**: Database change, no code, immediate
2. **Customer has agency**: They decide when to change their API (no dependency on Apsis backlog)
3. **Fallback position**: If it doesn't work, Solution 2 is still available for future investment
4. **Manageable risk**: Risks are understood and can be mitigated with documentation and clear expectations

### What NOT to Present

[Lukasz Grabowski]: We should not propose Solution 1 to the customer. It's too complex and would signal that we're considering a major platform change for one customer.

Keep Solution 1 internally for reference only.

---

## Next Steps and Meeting Preparation

### Immediate Actions

1. **Schedule meeting with Apsis internal stakeholders**: Alexander Linczkok (and optionally Opri)
2. **Prepare presentation materials**: Screenshot/diagram of the problem scenario
3. **Align on recommendation**: Confirm that Solution 3 is the proposal to take to the customer (Intermail/GolfAmore)

[Lukasz Grabowski]: We should have a chat with Alexander and Opri because Alexander wanted Opri involved in this discussion.

### Meeting Details

- **Date**: Tomorrow (January 22, 2026)
- **Time**: 3 PM
- **Attendees**: Erik Andersson, Lukasz Grabowski, Alexander Linczkok (required), Opri (optional per Alexander's call)
- **Duration**: 1 hour (Erik estimates 30 minutes won't be enough)
- **Topic**: "GolfAmore — Intermail Customer — Keyspace Email Identifier Challenge"

### Agenda Items

1. Present the three viable solutions (not Solution 1)
2. Get Alexander/Opri's perspective on:
   - Whether other customers have similar requests
   - Product strategy around keyspace customization
   - Effort estimates from their side (if Solution 2 is chosen)
3. Agree on communication strategy for the customer
4. Clarify risks (existing profiles, reinstallation, consent updates)

### Documentation Before Meeting

[Erik Andersson]: I'll share the whiteboard notes (diagram showing the problem scenario).

This diagram should be included in the Confluence documentation for the meeting and future reference.

---

## Key Technical Concepts Preserved

### Keyspace Fundamentals

A **keyspace** is a named space where profiles are keyed by a unique identifier. Each integration installs with a default keyspace:

- **Join CX**: Primary keyspace is `member`, keyed by CRM ID
- **E-deal** (Efficy Corporate): Profiles keyed by person ID
- **Efficy Enterprise**: Profiles keyed by person ID

The same contact can exist in **multiple keyspaces** with different keys (e.g., form submission creates profile in `form` keyspace, merge links it to `person` keyspace in Efficy).

### Installation Mapping Table

For each integration installation, a database table tracks:
```
installation_id → entity_type → keyspace_id
```

This allows the connector code to ask: "For this installation, when I receive an update for entity X, which keyspace should I write to?"

### Profile Merge Worker Use Cases

The merge worker is used when:
1. **Form submission**: Profile created in form keyspace, CRM creates entity, merge links them
2. **Lead to contact promotion**: Customer tells us a lead ID and contact ID are the same, merge them
3. **Duplicate detection**: Customer identifies two contacts are duplicates, merge them
4. **Cross-keyspace operations**: Any scenario where a profile must be linked across keyspaces

---

## Caveats and Warnings

### Email as Keyspace Key: Non-Negotiable Constraints

If email is used as a keyspace key:
- **Uniqueness is mandatory**: No two profiles can share the same email in that keyspace
- **Format validation is required**: All CRM ID values must be valid email addresses
- **Consent updates complexity**: The customer must send emails in `record_id` fields, not CRM IDs

This is not a workaround; it's a system constraint.

### Reinstallation Risk (Solution 3 Only)

If the customer reinstalls the integration:
- The database mapping reverts to default (join_cx keyspace)
- The email keyspace workaround is lost
- Integration breaks until the database change is re-applied

**Mitigation**: Document this clearly and consider a monitoring alert if the mapping gets reset.

### Orphaned Profiles Scenario (Solution 3 Only)

Existing profiles in the join_cx keyspace will not automatically migrate to the email keyspace. The customer will have two separate sets of profiles:
- **Old**: join_cx keyspace (keyed by old CRM IDs)
- **New**: email keyspace (keyed by email)

This duplication is a consequence of the workaround, not a defect.

### Consent/Subscription Updates Require Customer Implementation

Regardless of solution chosen, consent updates remain a customer responsibility. They must send email addresses in the API, not CRM IDs.

[Erik Andersson]: There's no pointing us doing anything manual on our side if we rely on them having that change on their side. They might as well do it for all of the updates.

---

## Why Solutions Were Evaluated This Way

### Philosophy: Effort vs. Purity

[Erik Andersson]:
> The beauty of developing is we can do anything. The question is always effort, priorities and philosophies.

Solution 1 is philosophically pure (full support for keyspace selection) but requires significant effort on both sides. Solution 3 is pragmatic (database workaround) but feels inelegant. Both are valid depending on organizational priorities.

### Prioritization Context

[Erik Andersson]: This is a request from one small customer for something that hasn't been requested before. I picture this being priority in 3-4 months ahead in time in a normal backlog.

This context explains why Solution 3 (immediate) is more attractive than Solution 2 (queued development).

### Pattern Recognition

[Erik Andersson]: They might already say "Oh yeah, but we have 20 other customers who want to do the same thing," or this could bring value.

Alexander and Opri have broader customer visibility. They might confirm whether this is a one-off edge case or an emerging pattern worth investing in.

---

## Key Takeaways

1. **The customer has a valid but unconventional use case**: Using email as the primary profile key is possible but requires system-level changes or workarounds.

2. **Four solutions exist with different trade-offs**:
   - **Solution 1** (configurable keyspace): Full redesign, high effort, reusable, complex
   - **Solution 2** (profile merge): Moderate effort, proper design, queued behind priorities
   - **Solution 3** (database mapping): Minimal Apsis effort, rapid, customer-driven timeline, documented workaround
   - **Solution 4** (CRM ID approach): No effort, customer rejects this philosophically

3. **Solution 3 is recommended for this customer** because:
   - It can be implemented immediately
   - It doesn't require Apsis development resources
   - The customer controls their own timeline
   - Risks are manageable and well-documented

4. **Solution 2 is the long-term insurance policy**: If other customers emerge with the same need, the merge approach becomes attractive for reuse.

5. **Critical customer requirements**:
   - Must always send email addresses in the `crmId` field (not their internal CRM ID)
   - Must send email addresses in `record_id` for consent updates
   - Must understand reinstallation will break the workaround
   - Should perform a full sync to migrate existing profiles or manually handle orphaned profiles

6. **Documentation and alignment first**: Before presenting to the customer, confirm with Alexander and Opri that Solution 3 is the right recommendation and whether this is a one-off or emerging pattern.

---

## Unresolved Questions and Action Items

### Pre-Meeting (Before January 22 3 PM)

- [ ] Erik: Prepare and share the whiteboard diagram (problem scenario visualization)
- [ ] Erik: Create detailed technical notes on the three solutions for the internal meeting
- [ ] Lukasz: Schedule meeting with Alexander Linczkok for 3 PM, invite Erik, note optional Opri

### In Meeting With Alexander/Opri

- [ ] Are there other customers asking for keyspace customization?
- [ ] Is cross-keyspace profile management a strategic direction?
- [ ] What is the effort estimate for Solution 2 from the product side?
- [ ] Should the database mapping table be exposed as a configuration option in the future?

### Pre-Customer Presentation

- [ ] Confirm Solution 3 is the recommendation
- [ ] Prepare slide deck with scenarios and risks
- [ ] Draft the database change and its documentation
- [ ] Plan migration strategy for existing profiles

### Post-Decision (If Solution 3 Is Chosen)

- [ ] Create Confluence documentation: "GolfAmore Email Keyspace Workaround"
  - Why it was implemented
  - Exactly what the customer must change in their API
  - Reinstallation mitigation steps
  - Rollback/recovery procedures
- [ ] Schedule follow-up with Intermail/GolfAmore
- [ ] Define SLA for the database mapping change (should it be monitored?)