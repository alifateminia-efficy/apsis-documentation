---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One - Integrations
topics: [Keyspace Configuration, Multi-Keyspace Profile Management, CRM Integration Patterns, Data Synchronization, Profile Merging, Email vs CRM ID Identifiers]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Join CX (Intermail), Apsis One, Keyspace, Profile Merge Worker, Integration Service, Database Configuration, Contact/Profile Entities]
session_type: knowledge-transfer
---

## Session Overview

This session discusses a critical architectural problem encountered with a specific customer integration between Join CX (an Intermail loyalty system) and Apsis One. The customer operates on email addresses as primary identifiers while the generic Join CX connector is designed around CRM IDs, creating a mismatch where the same profile exists in two separate keyspaces with no automated bridge between them. The discussion explores four potential solution approaches, ranging from full platform redesign to database-only workarounds, with emphasis on effort vs. implementation complexity.

---

## Problem Statement: Multi-Keyspace Identity Mismatch

### The Integration Architecture

The customer setup involves three systems in a specific relationship:

- **End Customer** uses the Join CX loyalty system (developed by Intermail) and interacts with it using **email addresses as their primary identifier**
- **Join CX** syncs data to **Apsis One** using the **generic connector**, which relies on **CRM ID identifiers** (not email)
- The end customer also has a **direct API connection to Apsis One**, again using **email addresses**—they have no knowledge of their CRM ID

[Erik Andersson]: > "The customer is also directly interacting with apps using one API and the customer has no idea what the CRM ID is. Their customer they only know using their e-mail."

### The Core Problem

When a profile is created via the Join CX sync, it exists in the **Join CX keyspace** with a CRM ID as the profile key. When the same customer interacts directly via API using their email, Apsis creates a duplicate profile in the **email keyspace** instead of finding and updating the existing one. This creates two critical failure modes:

1. **Data reads fail**: If they query by email, they get no results for the profile that was synced via CRM ID
2. **Duplicate profiles**: Any update via email creates a second profile instead of updating the original

[Erik Andersson]: > "So whatever their use case, it will either create duplicates or return no matching data."

---

## Solution Option 1: Keyspace Configuration During Installation (Complex, Not Recommended)

This approach would allow customers to select which keyspace the integration should use at setup time.

### Implementation Concept

Add a UI dropdown during installation asking: "Which keyspace should this integration utilize?" The generated connector code would then use the selected keyspace instead of the hardcoded one.

### Critical Problems with This Approach

**Unique Identifier Validation**: The fundamental constraint is that each keyspace has different rules for what constitutes a valid identifier:
- **CRM ID keyspace**: Can be any arbitrary string
- **Email keyspace**: Must be a valid email address

[Erik Andersson]: > "The unique identifier would also need to be configured for it because for the CRM ID for the CRM key space like you can specify essentially anything. It can be any string but for the e-mail key space it must be a valid e-mail address."

This means the connector code cannot simply extract a CRM ID; it must extract **email addresses** instead. The data extraction logic changes fundamentally.

**API Contract Changes Required**: For **consent/subscription updates**, the customer would need to send the **email address in the record ID field** instead of the CRM ID. This requires changes on their side and validation on ours.

**Mutually Exclusive Constraint**: If a customer needs the ability to have multiple profiles with the same email address (e.g., for testing or multi-account scenarios), that becomes impossible. 

[Lukasz Grabowski]: > "If there is a key, it's an e-mail. So it's just in one e-mail for one person. It's cannot be like multiple. It's you must be unique."

[Erik Andersson]: > "Like they are mutually mutually exclusive and that I hope they understand."

### Effort Assessment

This would require significant work on both sides (Apsis platform redesign + customer API changes) and represents a departure from the intended design pattern. **Not recommended for a single customer use case.**

---

## Solution Option 2: Profile Merge Worker Post-Creation (Moderate Effort)

This approach leverages existing merge infrastructure to automatically create a link between the CRM keyspace and email keyspace profiles.

### How It Works

After a profile is created in the **Join CX keyspace** (via the normal sync flow with CRM ID), automatically trigger a **merge operation** that:

1. Extracts the email address from the newly created profile
2. Sends a merge message to the `integration-profile-merge-worker` service
3. Merges the CRM-keyed profile with an email-keyed profile (creating one if it doesn't exist)
4. Subsequent queries by email return the same unified profile

[Erik Andersson]: > "If you select e-mail key space, we need to look at the e-mail addresses instead. So like instead of trying to download profiles and extract the CRM ID, we need to download contacts and extract the e-mail address."

### Merge Worker Context

The merge worker is an existing service primarily used for **form submission handling**:

[Erik Andersson]: > "When you submit the form this is created like in your. I don't know if you have the form key space or the e-mail and SMS key space when it is submitted when we so when we we receive that event from audience that this profile had has a form event submit, we will send that to the CRM system with all of the relevant event data they'll they will then respond saying. We created like this specific entity from it. Let's say that they create this lead entity for ED. They created a select from it. So we receive a ID back. We then do a merge operation."

Common use cases for merge:
- Form submission creates a profile in form keyspace → merge with CRM contact keyspace
- CRM system reports duplicate contacts merged → merge those profiles
- CRM system promotes a lead to contact → merge the lead profile with contact profile

### Implementation Details

**First Iteration (Hardcoded Approach)**: Since this is for a single customer, hardcode the email keyspace ID:

1. In the `profile-update-service`, after creating/updating a profile for this specific customer ID:
   - Extract the email attribute from the profile
   - Create a merge message with:
     - **Source keyspace**: Join CX keyspace (already have CRM ID)
     - **Source key**: CRM ID
     - **Destination keyspace**: Email keyspace ID (hardcoded for this customer)
     - **Destination key**: Extracted email address
   - Send to merge worker

2. If no email exists on the profile, skip the merge step

[Erik Andersson]: > "You would need to extract the ID of the key space but but but we have no no we have code for that already. So like we we have libraries."

[Lukasz Grabowski]: > "if this is this solution will be only for one customer, you are right, we can hard code it his ID like it will be very not elegant, elegant solution, but we can do do them for some time at least."

**Future Enhancement**: Convert to conditional configuration with UI option once pattern is validated with multiple customers.

### Effort Assessment

[Erik Andersson]: > "that would take you like 2-3 days. Maybe it wouldn't be that much."

**Estimated effort: 3-5 days** to implement. No changes required on customer side (they continue sending data as normal).

### Risks and Caveats

- **Dependency on email attribute**: Profile must contain valid email; merges are skipped for profiles without email
- **Duplicate cleanup**: Old profiles in Join CX keyspace are **not automatically deleted**—customer may see duplicates until they run a full re-sync
- **Reinstallation risk**: If customer reinstalls the integration, the merge logic remains but keyspace configuration reverts to normal, potentially breaking future syncs

---

## Solution Option 3: Database Configuration Change (Recommended, Minimal Effort)

This pragmatic workaround bypasses code changes entirely by modifying the integration's keyspace mapping in the database.

### Architecture Context: Integration Installation Mapping

When an integration is installed, Apsis creates a **keyspace mapping table** that stores which keyspace should be used for each entity type:

```
[Installation for Join CX customer]
Entity Type: member
Keyspace: join_cx_keyspace_12345
```

When the integration receives an update for a "member" entity, it looks up this mapping and applies the update to the specified keyspace.

[Erik Andersson]: > "So we have that mapping, so for this FC corporate instance, if we receive a update for a person, then we will do all the updates in this like we will utilize this key space discriminator for that."

### The Workaround

Instead of code changes, **directly modify the database mapping table** to point the integration to the email keyspace:

```
[Installation for Join CX customer]
Entity Type: member
Keyspace: email_keyspace  ← Changed from join_cx_keyspace_12345
```

**Effort**: 5 seconds of database modification. Can be done immediately.

### The Trade-off: Customer Must Send Email as CRM ID

The critical caveat: Apsis will still **look for the CRM ID field as the profile key identifier**, but the update will happen in the **email keyspace**. This only works if:

[Erik Andersson]: > "We will still look for the CRM ID as the identifier, but as we mentioned in the beginning, this will not work if we try to update the e-mail key space. I tried doing that yesterday, so we would still need. We would need the e-mail addresses provided to us as the CRM ID."

**The customer must send the email address in the CRM ID field** in all API requests:

- Profile updates: Send email in CRM ID field
- Consent updates: Send email in record ID field (not CRM ID)
- All future syncs: Consistently send email instead of their internal CRM ID

### Pre-Implementation Checklist

Before applying this change, the customer must **commit to sending email addresses** in place of CRM IDs everywhere they interact with the API. This is explicitly required because:

[Erik Andersson]: > "for consent updates they don't send us any e-mail attributes, they just give us CRM ID. So so we can't fix this for consent. We can map it like we will rely on them having that change on their side."

### Critical Risk: Existing Profile Duplication

**Major gotcha**: If the customer already has existing profiles synced in the **Join CX keyspace**, changing the keyspace mapping will **not migrate them**. Going forward, new profiles appear in the **email keyspace**, leaving old profiles orphaned in the Join CX keyspace.

[Erik Andersson]: > "If they already have existing contacts, they would exist of course in the join CX key space. If we change which key space, everything would be duplicated unless they do manual merge work before or they just delete every profile they have that is. Of course, also an option."

**Options to handle existing data**:
1. **Manual merge**: Customer or Apsis manually merges old profiles with new ones in the email keyspace
2. **Delete and resync**: Customer accepts data loss and triggers a full resync of all contacts (they will be created fresh in email keyspace)
3. **Accept duplicates**: Acknowledge that old data remains in join_cx keyspace while new data flows to email keyspace (not recommended but possible)

[Lukasz Grabowski]: > "So we need to also point point out this that about this possibility that they will have duplicated profile one in the e-mail key space and the 2nd in the CRM key space."

### Reinstallation Risk

If the customer ever **reinstalls the integration** (e.g., disconnecting and reconnecting), the setup process will **revert the database mapping to the default Join CX keyspace**. The change must be reapplied.

[Erik Andersson]: > "the caveat here is of course if they for whatever reason were to reinstall the installation, then this change must again be done on the side because we will have reverted to the normal joint CX in the database because that's that's. Set up on installation."

**Mitigation**: Document this as a known configuration and include in runbooks.

### Effort and Timeline Assessment

- **Apsis side**: 5 seconds (one database update)
- **Customer side**: Must modify their API integration to send emails instead of CRM IDs (varies by complexity)
- **Timeline advantage**: Customer can start changes immediately; doesn't depend on Apsis product/dev priorities

[Erik Andersson]: > "They can do this like in the next week. That is not. They don't need to sit and wait for you to finish your business priorities, which might be finished like in three months time."

### Why This Solution Over Option 2

For a single customer with an exceptional use case:

- **No platform changes** required (preserves integrity of intended design)
- **Immediate deployment** (no development cycle)
- **Puts control with customer** (they decide when/if to make their changes)
- **Transparent and documentable** (database change is explicit and auditable)

[Erik Andersson]: > "Me personally, I am a big proponent of this. Option. If anything had to be done on Appsys side, I would go for the merge one."

---

## Solution Option 4: No Changes—Use Intended Design (Not Viable)

The customer could choose to **stop using email identifiers** and instead:

1. Use the Join CX keyspace and CRM ID exclusively
2. Build custom logic in their system to map between customer email and CRM ID
3. Make their direct API calls to Apsis using CRM IDs instead of email

[Erik Andersson]: > "the customer utilizes the CRM ID and the join CX key space in their custom solution is that meaning they would need to essentially. Actually do the work between the join CX instance instead."

**Assessment**: This defeats the purpose of their request and requires them to fundamentally change how they've architected their system. **Not recommended** unless the customer explicitly prefers this burden.

---

## Recommended Path Forward: Option 3 with Option 2 as Backup

### Proposed Approach

**Primary recommendation**: Solution 3 (database configuration workaround)

- **Why**: Minimal effort, immediate deployment, transparent, non-invasive to platform design
- **Customer action required**: Modify API calls to send email addresses in CRM ID fields
- **Risk mitigation**: Document the change; warn about reinstallation; clarify the prerequisite

**Fallback if customer declines**: Solution 2 (merge worker automation)

- **Why**: Works with their existing API calls; requires Apsis dev effort but no customer API changes
- **Effort cost**: 3-5 days; subject to product priorities
- **Timeline**: Realistic 3-4 month horizon depending on backlog

### Next Steps: Internal Alignment Meeting

[Lukasz Grabowski]: > "OK, so I think we should have a chat with Alexander and Opri, maybe because Alexander said that he would like to. Have Opri also in this discussion."

A follow-up meeting is scheduled with **Alexander Lindström** (Product/Architecture) and optionally **Opri** (subject matter expert) to:

1. **Validate the analysis**: Confirm no design flaws in proposed solutions
2. **Check for precedent**: Determine if other customers have similar requirements (would shift priority toward Option 2)
3. **Clarify ownership**: Confirm expectations on which side (Apsis vs. customer) should own the work
4. **Document caveats**: Ensure all risks (reinstallation, duplication, email validation) are communicated to the customer

**Meeting scheduled**: Tomorrow (January 22, 2026) at 3 PM, 1 hour duration

**Meeting title**: "Intermail / Golfam Re-Keying Solution Alignment"

**Attendees**: Erik Andersson, Lukasz Grabowski, Alexander Lindström (+ Opri if available)

---

## Key Takeaways

1. **The problem is real and non-trivial**: A single customer cannot use email as a profile key while the integration is designed around CRM IDs, creating a fundamental architectural mismatch.

2. **Four solutions exist, each with trade-offs**:
   - **Option 1** (Full redesign): Comprehensive but expensive and breaks existing design patterns
   - **Option 2** (Merge automation): Clean long-term solution; 3-5 days effort; no customer changes needed
   - **Option 3** (Database hack): Immediate 5-second fix; requires customer to send emails instead of CRM IDs; small reinstallation risk
   - **Option 4** (No change): Customer adapts; not recommended as it negates their request

3. **Solution 3 is pragmatic for a one-off case**: If this is truly a single customer, the database workaround is low-risk and fast. If multiple customers want this, Option 2 becomes the right choice.

4. **Customer prerequisites are non-negotiable**: Whatever solution is chosen, the customer must understand that they need to send email addresses consistently and cannot support multiple profiles per email.

5. **Documentation is critical**: Any workaround must be explicitly documented (especially the reinstallation gotcha) and included in integration runbooks.

6. **Product alignment needed**: Before committing to any solution, Apsis internal stakeholders (product, architecture) must weigh this against:
   - Other customers with similar requests
   - Product roadmap and priorities
   - Philosophy on supporting non-standard keyspace usage

---

## Unresolved Questions and Action Items

### Questions Pending Internal Discussion

- **Precedent check**: Are there other customers requesting multi-keyspace support or email-as-key functionality?
- **Scope creep risk**: If we solve this for Join CX, will other integration partners request the same capability?
- **Long-term design**: Should Apsis officially support keyspace selection at installation time, or maintain the current one-keyspace-per-integration model?

### Action Items

| Item | Owner | Deadline | Status |
|------|-------|----------|--------|
| Schedule internal alignment meeting with Alexander & Opri | Lukasz Grabowski | January 22, 2026, 3 PM | Scheduled |
| Share solution documentation and diagrams with internal team | Erik Andersson | Before 1/22 meeting | In progress |
| Determine customer's existing profile status (are profiles already synced?) | *To be assigned* | Before customer call | Pending |
| Prepare customer-facing proposal with all four options and risks | *To be assigned in meeting* | TBD after 1/22 | Pending |
| If Option 3 chosen: Create runbook for database modification and reinst allation recovery | *To be assigned in meeting* | TBD | Pending |
| If Option 2 chosen: Scope detailed implementation plan for merge worker integration | *To be assigned in meeting* | TBD | Pending |

---

## Technical Reference: Database Mapping Table Structure

[Based on Erik Andersson's explanation of the integration configuration system]

```
Integration Installation Mapping Table:
├── Installation ID: <unique per customer>
├── Integration Type: "join_cx" | "form" | other
├── Entity Type: "member" | "contact" | "lead" | etc.
├── Keyspace ID: <keyspace identifier>
└── [Additional configuration fields]

Example - Default Join CX Setup:
Installation: customer-12345
Entity: member
Keyspace: join_cx_keyspace_12345

Example - Database Workaround (Option 3):
Installation: customer-12345
Entity: member
Keyspace: email_keyspace  ← Modified from join_cx_keyspace_12345
```

The integration service queries this mapping table on every incoming update to determine the target keyspace.

---

## Session Metadata

**Duration**: 53 minutes 6 seconds

**Key Diagrams Referenced**: Whiteboard drawing comparing the three-system integration (End Customer → Join CX → Apsis) with emphasis on identifier mismatch. Requested to be captured and added to Confluence documentation.

**Document References**: Solution notes shared with Lukasz Grabowski (shared access attempted; access granted via personal invite at end of session).