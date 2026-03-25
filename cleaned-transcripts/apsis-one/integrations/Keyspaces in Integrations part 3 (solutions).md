---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One Integrations
topics: [Keyspace Configuration, Multi-Identifier Profile Management, Integration Architecture, Merge Operations, CRM ID vs Email Identifier Resolution]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Apsis One, Join CX (Intermail), Generic Connector, Keyspace Mapping, Profile Merge Service, Integration Profile Update Service]
session_type: architecture-review
subdomains: [Architecture, Microsoft Dynamics Integration]
---

## Session Overview

This knowledge transfer session explores three distinct solutions for a complex integration problem where a Join CX customer (Intermail) needs to use email addresses as unique identifiers in Apsis One, rather than CRM IDs. The customer operates across two systems simultaneously—Join CX (which syncs via CRM ID to Apsis) and a direct API integration (where they only know email addresses)—creating a mismatch that leads to duplicate profiles or missing data. Erik and Lukasz evaluate each solution's trade-offs in terms of implementation effort, risk, and feasibility, ultimately recommending Solution 3 (database mapping change with customer-side ID field modification) as the most pragmatic approach for this single-customer edge case.

---

## The Integration Problem: Dual Access Patterns and Identifier Mismatch

### Customer Setup and Context

The end customer uses **Join CX** (a customer loyalty system developed by Intermail) and interacts with it exclusively via **email addresses** as identifiers. This same customer also maintains a **direct integration with Apsis One via API**, but their internal systems only know customers by email addresses, not CRM IDs.

[Erik Andersson]: The customer is integrated with Join CX, and they interact with their system using only email addresses. Join CX syncs to Apsis using a CRM ID identifier through the generic connector. But the end customer also directly interacts with Apsis via one API, and the customer has no idea what the CRM ID is—they only know their customers by email address.

### The Core Problem: Two Incompatible Profile Namespaces

The integration creates profiles in the **Join CX keyspace** (keyed by CRM ID) when syncing from Intermail. However, the customer's direct API calls attempt to create or update profiles in the **email keyspace** (keyed by email address), not knowing about the CRM ID keyspace.

[Lukasz Grabowski]: So we have a CRM profile created in the CRM keyspace, and it has an email attribute. But they use the email as a profile key—that's the problem. From a high level, it looks like there's no mapping between the two access patterns.

This creates two failure scenarios:

1. **No data found**: When the customer queries via email address, they find no profile (it exists only in the CRM keyspace under a CRM ID).
2. **Duplicate profiles**: When the customer attempts to update a profile via email, a new duplicate profile is created in the email keyspace instead of updating the existing one in the CRM keyspace.

[Erik Andersson]: Whatever their use case, it will either create duplicates or return no matching data... If they try to update by adding a secondary mobile number, a second duplicate profile will be created in the email keyspace instead of the CRM keyspace.

---

## Solution 1: Full Keyspace Configuration System (Not Recommended)

### Concept

Allow customers to specify which keyspace an integration should use at installation time via a dropdown menu. Instead of always bootstrapping the Join CX keyspace with CRM IDs, the customer could select "email keyspace" and the generated connector code would use email addresses as the unique identifier throughout.

### Implementation Requirements and Complications

This solution is **conceptually simple but practically complex** because it requires fundamental changes to how the connector extracts and validates identifiers:

- The unique identifier validation is **keyspace-dependent**. CRM ID keyspace accepts any string value, but the **email keyspace requires a valid email address format**.
- The connector must be reconfigured to extract email attributes instead of CRM IDs from the source system.
- For consent and subscription updates, the customer would need to send email addresses in the `record_id` field instead of CRM IDs—a non-trivial API change on their side.

[Erik Andersson]: The unique identifier would need to be configured differently because for the CRM ID keyspace, you can specify essentially anything—any string. But for the email keyspace, it must be a valid email address. So the code cannot look for the CRM ID or contact ID on a profile; it must extract the email address instead.

### Critical Constraint: Mutually Exclusive with Multi-Email Profiles

Using email as a key means **one email per profile**—multiple profiles with the same email address become impossible. This is not a configuration option; it is a hard constraint of keyspace semantics.

[Lukasz Grabowski]: If there is a key, it's email. So it's just one email for one person. It cannot be like multiple profiles. You must be unique. So this is impossible, right?

### Verdict

Both Apsis and Intermail would need to invest significant effort (estimated weeks), and the solution would require agreement that multiple profiles per email are not needed. **Erik and Lukasz agreed this is "rather less possible to do."**

---

## Solution 2: Merge-Based Synchronization (Moderate Effort, Apsis-Side)

### Concept

Rather than changing the keyspace configuration, maintain the existing Join CX → CRM ID keyspace sync, but **automatically merge** the resulting profile into the email keyspace after creation/update. This leverages the existing **profile merge worker** (integration profile merge service), which is already used in form submission workflows.

### How It Works

1. Customer sends a create/update request with a profile (identified by CRM ID).
2. Apsis creates or updates the profile in the Join CX keyspace as normal (using CRM ID).
3. A message is automatically sent to the **integration profile merge worker** service, instructing it to merge the same profile into the email keyspace using the email address extracted from that profile.
4. Result: The profile now exists in **both keyspaces** and can be found via either CRM ID or email address.

[Erik Andersson]: If you want to have this full-fledged, in the future you can have the UI saying "also merge with this keyspace" and you can have that as a conditional check. In the first implementation, you could just do this in the code—if it is for this specific customer, then add a merge message.

### Implementation Details

- The merge service already exists and is used for form submission handling (merging form keyspace profiles with CRM keyspace profiles).
- For the first iteration, the email keyspace ID could be **hard-coded** since this is a single-customer solution.
- Libraries to extract keyspace IDs are already available in the codebase.
- Estimated effort: **2-3 days of work**.

[Erik Andersson]: This would take you maybe 2-3 days. Maybe it wouldn't be that much. You don't need to build the spaceship right now. You can do this as a specific implementation for this customer.

### Merge Service Mechanics (Context from Form Submissions)

[Erik Andersson]: When you submit a form, we receive that event in Apsis that a profile had a form event submission. We send that to the CRM system with all relevant event data. They respond saying "we created this lead entity from it." We receive an ID back, and then we do a merge operation. We merge this profile that exists in the form keyspace with the Join CX keyspace so the profile can be found and we don't create duplicates. Also, the CRM system can send us merge requests—they can say "we accidentally created a duplicate of these two contacts, so this profile with CRM ID X and CRM ID Y are actually the same."

### Advantages

- Minimal effort on Apsis side (2-3 days).
- No changes required to customer's API calls or field mapping.
- Leverages proven, existing infrastructure (merge worker).
- Full-fledged UI feature could be built later.

### Disadvantages

- Still requires Apsis engineering work (subject to priority backlog).
- Work is deferred from customer to Apsis internal team.
- Creates two copies of each profile (one per keyspace), consuming more storage.

### Verdict

**Viable but subject to internal priorities.** Erik suggested this would be preferable if Apsis needed to do any server-side work, but noted this is a 3–4 month priority estimate for a small customer with a non-standard use case.

---

## Solution 3: Database Mapping Change with Customer-Side Identifier Swap (Recommended - Lowest Apsis Effort)

### Concept

Apsis maintains an **internal mapping table** that tracks which keyspace each integration should use for a given entity type. For example:

```
Integration: Join CX / Customer
Entity Type: Member
Keyspace: join_cx_keyspace_v1
```

**Solution 3 modifies this mapping** to point to the email keyspace instead:

```
Integration: Join CX / Customer
Entity Type: Member
Keyspace: email_keyspace
```

This is a **5-second database change**. All downstream logic automatically uses the new keyspace without any code modifications.

[Erik Andersson]: We would just go into the database and change this to the email keyspace. Then the integration, whenever there is an update for member, will try and do those updates on the email keyspace instead. And this is without changing any code or logic on the Apsis side.

### The Critical Requirement: Customer Must Provide Email as CRM ID

**This solution places the burden on the customer.** Because Apsis still looks for the **CRM ID field** as the unique identifier, the customer must send email addresses in the `CRM ID` field for all requests (profile updates, consent updates, etc.).

This is not a "hack" in the sense of being wrong; rather, it is **redefining what the "CRM ID" field means for this specific integration**: it now contains email addresses instead of CRM identifiers.

[Erik Andersson]: This solution will require work from them because we will need them to send us the email address value in the CRM ID field instead. If they do that, then everything is fixed for the customer because then they will update everything via the email keyspace.

### Implementation Steps

1. **Customer notifies Apsis**: "We are ready to send email addresses in the CRM ID field for all requests."
2. **Apsis updates database mapping** (5 seconds, 1 SQL change).
3. **Ongoing requirement**: Every profile update, every consent update, every API call must send the email address in the `CRM ID` field—not an actual CRM ID.

### Key Gotchas and Risks

#### Reinitialiation Risk

If the customer **reinstalls the integration**, the mapping will revert to the default (join CX keyspace with CRM ID). The customer and Apsis must remember to reapply the database change.

[Erik Andersson]: The caveat here is that if they for whatever reason were to reinstall the installation, then this change must again be done on the Apsis side because we will have reverted to the normal join CX in the database.

**Mitigation**: This is a single-customer edge case, so reinstalls should be rare. Internal documentation and a note in the integration's Confluence page will help.

#### Existing Profile Duplicates

If the customer already has profiles synced under the CRM keyspace (with actual CRM IDs), switching to the email keyspace will create **duplicate profiles** in the email keyspace when new syncs arrive.

[Lukasz Grabowski]: What will happen with old profiles? For example, they have some profiles in the CRM key space. And now the same profile will be synced to us with email keyspace and email address as key. So then what will happen?

[Erik Andersson]: All their old data will exist in the CRM keyspace as is. Now, if we implement this logic to merge after each profile update, they will only have one instance if they do a full sync. Otherwise, they will have duplicates unless they do manual merge work before or just delete every profile.

**Mitigation**: The customer must choose one of:
1. Run a full sync of all contacts after the database change is applied (Apsis can do this; each update will now use email keyspace and can be merged).
2. Manually merge old CRM keyspace profiles with new email keyspace profiles.
3. Accept the duplication (not ideal).

### Advantages

- **Minimal Apsis effort**: 5-second database change, can be done immediately.
- **No code changes**: No logic, no configuration, no deployment needed.
- **Customer self-service**: Can be implemented in parallel with other work; customer controls timing.
- **Low risk for future iterations**: If this becomes a broader requirement, Solution 2 can be built later as a permanent feature.

### Disadvantages

- Customer must modify their API contract (send emails in CRM ID field).
- All consent updates must also send emails in record ID field (not just CRM ID as usual).
- Requires customer coordination and testing.
- Database manual change must be re-applied if reinstall occurs.

### Verdict

**Recommended for immediate implementation.** This is pragmatic for a single-customer edge case. Erik: "Personally, I am a big proponent of this option."

---

## Solution 4: No Change—Maintain Design-as-Intended (Not Recommended)

### Concept

Do nothing. The integration was designed to use CRM IDs. The customer should adapt their architecture to use CRM IDs instead of email addresses, or handle the mapping externally (e.g., in Join CX or in a middleware layer).

[Erik Andersson]: The customer utilizes the CRM ID and the join CX keyspace. In their custom solution, they would need to essentially do the work between the join CX instance themselves.

### Verdict

**Possible but unlikely to be accepted by the customer.** This preserves system design integrity but shifts responsibility entirely to the customer, who may lack the technical capacity or willingness to do so.

---

## Recommended Path Forward: Solution 3 + Solution 2 Evaluation

### Immediate Actions (Next 24 Hours)

1. **Schedule meeting with Alexander Linzkok and Opri** (Apsis product/architecture team) to discuss solutions and prioritize any Apsis-side work.
   - Meeting scheduled: Tomorrow (January 22) at 3 PM
   - Duration: 1 hour
   - Attendees: Erik Andersson, Lukasz Grabowski, Alexander Linzkok, (Opri optional per Alexander's discretion)
   - Customer: Intermail (Golf Amore partnership)

2. **Present to internal team**:
   - Solution 1 (full config system): Document for future reference but not proposed to customer.
   - Solution 2 (merge-based): Propose as a potential feature if other customers have similar needs; estimate 2–3 days of effort.
   - **Solution 3 (database mapping)**: Propose as immediate workaround; 5-second deployment on Apsis side, customer changes required.
   - Solution 4: Acknowledge as baseline (system works as designed).

3. **Identify blockers before customer discussion**:
   - Do existing profiles already exist in the CRM keyspace for this customer, or is this a greenfield sync?
   - If existing profiles exist, what is the merge strategy?
   - Are there other customers requesting similar functionality?

### Customer Engagement (Post-Internal Alignment)

1. **Present Solution 3 as primary recommendation**:
   - Apsis can apply the database change immediately upon customer confirmation.
   - Customer must modify their API calls to send email addresses in the CRM ID field.
   - Document the change and reinstation risk.

2. **Mention Solution 2 as a future alternative** if customer prefers Apsis-side implementation (estimate 2–3 months priority queue).

3. **Get written agreement** on:
   - Customer will send email addresses in CRM ID field for all requests.
   - Customer will perform a full sync after the database change to avoid duplicates.
   - Reinstation risks are understood.

### Documentation Requirements

- Confluence page documenting this edge case and solution.
- Internal code comment flagging the customer-specific keyspace override.
- Runbook for reapplying the database change if reinstallation occurs.

---

## Key Takeaways

1. **The core problem is architectural**: The system was designed for one identifier pattern (CRM ID per keyspace), but the customer needs to access profiles via two different identifiers (CRM ID and email) without maintaining that mapping themselves.

2. **Four solutions exist, with different trade-offs**:
   - Solution 1 (full config): High effort, supports future flexibility, but impossible if multi-email profiles are needed.
   - Solution 2 (merge): Moderate effort (2–3 days), uses proven infrastructure, but defers to Apsis backlog.
   - Solution 3 (database swap): Minimal Apsis effort (5 seconds), shifts burden to customer, but pragmatic for edge cases.
   - Solution 4 (no change): Preserves design, but likely unacceptable to customer.

3. **Solution 3 is recommended for immediate implementation** because:
   - It requires no Apsis code changes and can be deployed in minutes.
   - It empowers the customer to self-serve (can implement in parallel with other work).
   - It serves as a proof-of-concept; if multiple customers request this, Solution 2 can be built as a proper feature.
   - It does not violate system semantics; it redefines the "CRM ID" field for this customer as their de facto unique identifier (email).

4. **Implementation is conditional on customer action**:
   - Customer must modify all API calls to send email addresses in the CRM ID field.
   - Customer must decide how to handle existing CRM keyspace profiles (merge, delete, or accept duplication).
   - Reinstation is a manual step that must be repeated if the integration is reinstalled.

5. **This is a valid but atypical use case**:
   - The customer is simultaneously using two integration paths (Join CX sync + direct API).
   - Email-only identifier access is not the system's native design.
   - The solution is acceptable precisely because it is a single-customer edge case; broader adoption would necessitate Solution 1 or Solution 2.

---

## Unresolved Questions & Action Items

### Before Internal Alignment Meeting (Tomorrow, 3 PM)

- [ ] **Erik**: Prepare visual diagram of the three solutions for the internal meeting.
- [ ] **Lukasz**: Confirm meeting with Alexander Linzkok and Opri; send invites.
- [ ] **Both**: Verify whether Intermail already has synced profiles in the CRM keyspace (impacts duplicate risk).

### During/After Internal Alignment Meeting

- [ ] **Alexander**: Assess whether other customers have similar multi-identifier needs (informs Solution 2 viability).
- [ ] **Product Team**: Prioritize Solution 2 if multiple customers are waiting; otherwise, confirm Solution 3 as sufficient.
- [ ] **Erik**: Prepare communication to customer with Solution 3 rationale and requirements.

### Customer Rollout (Post-Alignment)

- [ ] **Customer**: Modify API integration to send email addresses in CRM ID field for all requests (profile updates and consent updates).
- [ ] **Customer**: Confirm readiness for database mapping change.
- [ ] **Lukasz/Erik**: Apply database mapping change once customer confirms.
- [ ] **Customer**: Perform full sync to migrate profiles to email keyspace and avoid duplicates.
- [ ] **Lukasz**: Document the edge case in Confluence with reinstation risk warnings.

### Documentation

- [ ] Add customer-specific note to integration configuration documenting the keyspace override and reinstation procedure.
- [ ] Include in internal runbook for future team members who may reinstall this integration.