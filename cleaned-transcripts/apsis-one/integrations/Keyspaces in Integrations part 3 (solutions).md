---
source_file: Keyspaces in Integrations part 3 (solutions).txt
domain: Apsis One Integrations
topics: [Keyspace Configuration, Generic Connector, Profile Merging, Customer Integration Workarounds, Email vs CRM ID Identifier Mapping]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Generic Connector, Join CX (Intermail), Profile Merge Worker Service, Keyspace Discriminator Mapping, Email Keyspace, CRM ID Keyspace]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow]
---

## Session Overview

This knowledge transfer session discusses a complex integration scenario where a customer (using Join CX loyalty system by Intermail) wants to interact with Apsis One using email addresses as profile identifiers, rather than the standard CRM ID approach. The team explores four potential solutions ranging from full system redesign to lightweight database configuration workarounds. The discussion emphasizes the tension between flexibility, effort, and architectural consistency, with the team ultimately recommending a pragmatic "database mapping hack" as the best path forward for this one-off customer scenario.

---

## The Customer Problem: Email vs CRM ID Identifier Mismatch

### Current System Architecture

[Erik Andersson]: The end customer uses a **Join CX customer loyalty system** developed by Intermail. In this system, they interact using email addresses as their primary identifier. Join CX then syncs data to Apsis One using a **CRM ID identifier** in the generic connector.

The flow looks like this:
- End customer system → uses email addresses
- Join CX platform → syncs to Apsis using CRM ID (via Join CX keyspace)
- Apsis One → receives and stores profiles in Join CX keyspace with CRM ID as key

### The Fundamental Issue

[Lukasz Grabowski]: The customer also directly interacts with Apsis One API, but only knows email addresses—they don't know what the CRM ID is. They want to use email addresses as their profile identifier in Apsis, not CRM IDs.

[Erik Andersson]: This creates a mismatch: the customer sends email addresses to Apsis API, but Apsis expects CRM IDs as the unique identifier for the Join CX keyspace.

### Impact: Duplicates and Data Loss

If the customer tries to update a profile using an email address when Apsis only recognizes it via CRM ID:
- **Profile updates via email will create duplicate profiles** in the email keyspace rather than updating the existing CRM-identified profile
- **Data queries using email addresses return no results** because the profile only exists under the CRM ID in the Join CX keyspace
- **Custom integrations fail**: For example, trying to read loyalty points by email returns no matching data

[Erik Andersson]: Whatever the customer's use case, it will either **create duplicates or return no matching data**. This is a fundamental problem.

---

## Solution Option 1: Allow Per-Installation Keyspace Selection (Complex, Not Recommended)

### The Concept

[Erik Andersson]: Allow customers to specify which keyspace the integration should utilize during installation, instead of hard-coding the Join CX keyspace for all installations.

**Implementation approach:**
```
Installation process:
  → Add dropdown: "Select which keyspace to use"
  → Customer selects: "Email Keyspace"
  → Generate connector code configured for email keyspace
  → Use conditional logic: if this customer, use email keyspace instead of CRM keyspace
```

### Critical Problems with This Approach

[Erik Andersson]: Even if we specify the keyspace, the **unique identifier configuration also needs to change**. For the CRM ID keyspace, the identifier can be any string. For the email keyspace, **it must be a valid email address**.

This means:
- We can't extract CRM IDs from profiles and use them as identifiers
- We must extract **email attributes** from contacts instead
- **Consent/subscription updates would require the customer to send email addresses instead of CRM IDs** in the record ID field

[Lukasz Grabowski]: There's also an architectural conflict: if multiple profiles with the same email address are needed, using email as a keyspace identifier **makes this impossible**. Email and phone keyspaces are designed for one-to-one mapping.

[Erik Andersson]: These use cases are **mutually exclusive**. If you want email keyspace support, you cannot have multiple profiles per email address.

### Effort Estimate

This solution requires changes on **both Apsis and the external system side**. Estimated effort: **significant** (timeline mentioned as 3-4 months priority queue).

### Why We Don't Recommend This

[Erik Andersson]: This is "very complicated" to implement. It requires rearchitecting fundamental assumptions about how the generic connector works.

---

## Solution Option 2: Automatic Profile Merge After Updates (Moderate Effort, Realistic)

### The Concept

[Erik Andersson]: Instead of hard-coding which keyspace to use at installation, we can keep the normal Join CX keyspace setup, but **automatically merge profiles** from the email keyspace after each profile update.

**Implementation flow:**
```
Customer sends profile update (with CRM ID):
  1. Profile Update Service creates/updates profile in Join CX keyspace (normal flow)
  2. NEW STEP: Extract email attribute from the CRM entity
  3. NEW STEP: Send merge message to Profile Merge Worker Service
     - Source: Join CX keyspace + CRM ID
     - Destination: Email keyspace + extracted email address
  4. Merge completes → profile is now accessible via both CRM ID and email
```

### How Profile Merge Worker Currently Works

[Erik Andersson]: The merge worker is already used for **form submission handling**. When a form is submitted:
1. Profile is created in form/email keyspace
2. Form submission triggers CRM to create an entity (e.g., a lead)
3. CRM returns the entity ID back to Apsis
4. Apsis merges: form keyspace profile ← → lead keyspace profile
5. Now both the form-originated profile and the CRM-created profile point to the same person

### Why This Works for the Customer

[Lukasz Grabowski]: If this merge happens, the customer can find the profile via email address. Duplicate email profiles will consolidate into a single profile through the merge operation.

[Erik Andersson]: The merge worker takes a source keyspace/key and a destination keyspace/key. We already have the source (CRM ID + Join CX keyspace). We just need to extract the destination (email + email keyspace ID).

### Implementation Effort

[Erik Andersson]: This is **considerably easier** than Option 1. Since merge logic already exists:
- Extract email from the CRM contact entity
- Create a merge message in the profile update service
- Add it as a step after profile creation
- **Estimated effort: 2-3 days of development**

[Lukasz Grabowski]: For a single-customer workaround, we could even hard-code the email keyspace ID rather than look it up dynamically, making it faster.

### Advantages

- Leverages existing merge infrastructure
- **Much lower effort** than Option 1
- **Work can happen on Apsis side** without blocking on customer changes
- Can be implemented as a specific customer workaround
- Scalable if other customers request the same feature later

### Disadvantages

- Requires the customer to have email addresses available on every contact/member entity
- If the customer reinstalls the integration, the merge logic doesn't auto-kick in on old profiles (only new updates trigger merges)

---

## Solution Option 3: Database Mapping Workaround (Fastest, Pragmatic)

### The Concept

[Erik Andersson]: Instead of changing code logic, we change the **database mapping table** that tells Apsis which keyspace to use for each integration.

**How it works:**
```
Integration Mapping Table (per installation):
  Join CX Installation A → Entity "Member" → Join CX Keyspace [CURRENT]
  Join CX Installation A → Entity "Member" → Email Keyspace [CHANGE TO THIS]
```

When Apsis receives an update for the "Member" entity type, it looks up the mapping table and uses whichever keyspace is configured—without any code changes.

[Erik Andersson]: This solution takes **5 seconds to implement**. Just change one database row and "everything in Apsis is essentially fixed."

### The Critical Catch: Customer Must Change Their API Payloads

[Erik Andersson]: The remaining problem is that **Apsis still looks for the CRM ID as the identifier**. If we switch to email keyspace without changing the identifier logic, it will fail because email keyspaces need valid email addresses, not arbitrary CRM IDs.

**Solution: The customer must send email addresses in the CRM ID field.**

```
Customer's current API payload:
{
  "record_id": "CUST123",     // CRM ID
  "email": "john@example.com"
}

Customer's new API payload (after they make changes):
{
  "record_id": "john@example.com",  // NOW: email address, not CRM ID
  "email": "john@example.com"        // still included, but not used as identifier
}
```

This applies to **all updates**: profile updates, consent updates, subscription changes, etc.

### Why This Works

[Lukasz Grabowski]: Once the customer sends email addresses in the `record_id` field, the integration workflow becomes:
1. Apsis receives update with email in CRM ID field
2. Database mapping sends it to email keyspace
3. Email keyspace accepts the email as a valid identifier
4. Everything works

**Both sides achieve their goals:** customer works with emails, Apsis doesn't need code changes.

### Pros

- **Minimal effort on Apsis side** (literally a database update)
- **Can be deployed immediately** — doesn't require Apsis release cycles
- **Customer maintains control** — they decide when to make their changes
- **Pragmatic for one-off scenarios**
- Does not redesign the system architecture

### Cons

- **Requires customer coordination** — they must change their API payloads
- **Breaking change if they reinstall:** If they reinstall the integration, the database mapping reverts to normal Join CX keyspace. They'd need the mapping changed again
- **Requires documentation:** This is a non-standard workaround and should be clearly flagged as such
- **All data migration pain falls on the customer:** If they already have profiles synced to Join CX keyspace under CRM IDs, those profiles won't automatically appear in email keyspace unless manually merged

### Data Migration Concern

[Lukasz Grabowski]: If the customer already has existing profiles in the Join CX keyspace (identified by CRM ID), switching the database mapping will create duplicates—one set in CRM keyspace, one set in email keyspace.

[Erik Andersson]: The customer has two options:
1. Perform a **full sync** where all existing contacts are re-downloaded and re-merged into email keyspace
2. **Delete all existing profiles** and start fresh with email-based profiles

Either way, this is **the customer's responsibility**, not Apsis's.

### Effort Estimate

- Apsis side: ~5 minutes (database update + documentation)
- Customer side: API payload changes (can be prioritized independently)

---

## Solution Option 4: Status Quo (No System Changes)

[Erik Andersson]: The customer can simply follow the system as designed: use CRM ID and Join CX keyspace. Their custom API integration would need to translate email addresses to CRM IDs on their side.

[Lukasz Grabowski]: This is technically valid since the system was designed around CRM IDs, not emails. However, it puts the translation burden entirely on the customer.

**Why it's unlikely to be accepted:** The customer specifically wants to avoid managing CRM IDs.

---

## Recommendation and Next Steps

### Recommended Approach: **Solution Option 3 (Database Mapping Workaround)**

[Lukasz Grabowski]: Given that this is a single customer with an exceptional use case, the database mapping hack is the best approach.

**Why:**
- Minimal disruption to Apsis architecture
- Can be deployed immediately without release cycles
- Customer has autonomy to schedule their API changes
- Clearly documentable as a workaround, not a precedent

### What to Do Before Moving Forward

[Erik Andersson]: We should **not present Option 1 to the customer**—it's overly complex and not pragmatic. We should focus on Options 3 and 2.

**Key points to clarify with stakeholders (Alexander and Opri):**

1. **Does the customer currently have existing profiles?** If yes, we need to address the migration/duplication problem
2. **Reinstallation risk:** If they uninstall/reinstall, the database mapping reverts. Needs to be documented as a risk
3. **Other customers:** Do others have this same need? This affects whether we should build Option 2 (merge-based solution) instead
4. **Timeline expectations:** Are they willing to coordinate API changes on their end, or do they expect a pure Apsis-side fix?

### Implementation Requirements for Option 3

**Apsis side:**
- Change database mapping table entry for this Join CX installation
  - Entity type "Member" → point to email keyspace instead of Join CX keyspace
- Hard-code the email keyspace ID in the mapping (it won't change)
- Add documentation flag to the installation record noting this is a non-standard configuration

**Customer side (Intermail):**
- Update their API integration to send email addresses in the `record_id` field
- Update consent/subscription API calls to use email instead of CRM ID
- Coordinate full data sync if existing profiles need to be migrated

---

## Key Takeaways

1. **The core issue is architectural mismatch**, not a bug: keyspaces are designed around one specific identifier (CRM ID for Join CX, email for email keyspace). The customer wants to use two systems without unifying on one identifier.

2. **Four solutions exist, each with different trade-offs:**
   - Option 1 (full redesign): Theoretically complete, but high effort and complexity
   - Option 2 (merge-based): Good for scaling to multiple customers, moderate effort
   - Option 3 (database mapping): Fast and pragmatic, customer-friendly, but non-standard
   - Option 4 (status quo): Technically valid, unlikely to satisfy customer

3. **Option 3 is recommended for this one-off case** because it's pragmatic, minimal on Apsis side, and empowers the customer to move forward independently.

4. **Key risk to document: reinstallation**. If the customer reinstalls, they'll need to request the database mapping change again. This should be prominently flagged in documentation.

5. **Data migration is the customer's responsibility.** Existing profiles in CRM keyspace won't auto-migrate to email keyspace. Apsis doesn't own fixing this.

6. **Before customer engagement**, internal discussion with Alexander (product) and Opri (engineering lead) is needed to confirm no architectural objections and assess if other customers might have this need.

---

## Unresolved Questions & Action Items

### Before Customer Meeting

- [ ] Confirm whether the customer currently has existing profiles synced from Join CX
- [ ] Check if other Intermail/Join CX customers have expressed this same requirement
- [ ] Clarify whether Option 2 (merge-based solution) should be built as a future feature if demand increases

### Next Internal Meeting (Scheduled for Tomorrow at 3 PM)

**Attendees:** Lukasz Grabowski, Erik Andersson, Alexander Linzkok, (possibly Opri)
**Topic:** Golf Amore / Intermail Customer Keyspace Integration Solutions
**Duration:** 1 hour
**Goal:** Agree on recommended solution and approach before customer discussion

**Meeting agenda:**
1. Review the four options and reasoning
2. Confirm Option 3 is acceptable from product perspective
3. Assess demand signal: are other customers requesting this?
4. Discuss if Option 2 (merge-based) should be scheduled for future work
5. Plan customer communication approach

### Before Customer Implementation

- [ ] Prepare written documentation of the workaround with risks and limitations
- [ ] Document the reinstallation caveat clearly
- [ ] Create database change script (updating keyspace discriminator for this installation)
- [ ] Brief Intermail on their required API changes and timeline
- [ ] Plan data migration strategy with customer (full sync vs. clean start)

---

## Technical Details for Future Reference

### Database Table Structure

Apsis maintains a **keyspace discriminator mapping table** per integration installation:

```
integration_installation_id | entity_type | keyspace_id
─────────────────────────────────────────────────────────
JOIN_CX_INSTANCE_001        | member      | JCX_KEYSPACE_ID
EDEALER_INSTANCE_001        | person      | EDEALER_KEYSPACE_ID
EDEALER_INSTANCE_001        | lead        | EDEALER_LEAD_KEYSPACE_ID
```

When an update arrives for an entity type, Apsis looks up this table to determine which keyspace to write to. For the workaround, we simply change the keyspace ID in this table.

### Profile Merge Worker Service Details

The merge worker is invoked with:
- **Source keyspace** + **source key** (what we currently have)
- **Destination keyspace** + **destination key** (what we want to merge into)

For this customer:
- Source: Join CX keyspace + CRM ID
- Destination: Email keyspace + email address extracted from the contact entity

The merge operation consolidates profiles and ensures queries work via either identifier.

### Email Keyspace Constraints

- **Identifier must be valid email address** (format validation enforced)
- **One profile per email address** (uniqueness enforced)
- Cannot have multiple profiles sharing the same email
- This is why "multiple profiles per email" is architecturally impossible