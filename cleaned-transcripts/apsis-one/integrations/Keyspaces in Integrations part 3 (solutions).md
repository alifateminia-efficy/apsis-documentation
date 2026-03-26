---
source_file: "Keyspaces in Integrations part 3 (solutions).txt"
domain: Apsis One Integrations
topics: ["keyspace configuration", "CRM ID vs email identifier", "profile merge", "integration workarounds", "duplicate profile prevention", "Join CX generic connector"]
speakers: ["Erik Andersson", "Lukasz Grabowski"]
key_components: ["Join CX", "Apsis One", "generic connector", "profile merge worker", "email keyspace", "CRM keyspace", "consent updates", "full sync"]
session_type: knowledge-transfer
subdomains: ["Duplicate profiles", "Different Types of Connectors"]
---

## Session Overview

Erik Andersson and Lukasz Grabowski work through a specific customer integration problem: an end customer using the Join CX (Intermail) generic connector is integrated with Apsis One via CRM ID keyspace, but the end customer's own system only knows email addresses — not CRM IDs. The session identifies the root cause of the conflict (two different keyspaces, two different identifiers, same person), evaluates four possible solutions in order of complexity, and settles on a database-level workaround (Solution 3) as the most pragmatic near-term fix. The session ends with scheduling a follow-up meeting with internal stakeholders (Alexander, Opri) and the customer contact (Max) to align on approach.

---

## Problem Setup: Conflicting Keyspaces and Identifiers

### The Three-Party Architecture

- **End customer** (named "Golf Amore"): operates using **email addresses** as their sole identifier when interacting with systems.
- **Intermail** (the CRM vendor): uses Apsis One via the **Join CX generic connector**. Because this is a generic connector, it operates on a **CRM ID identifier** and a dedicated **Join CX keyspace** that is bootstrapped at installation time.
- **Apsis One**: receives synced contacts from Join CX (stored under the CRM ID keyspace) and also receives direct API calls from the end customer (who only knows email addresses).

### The Core Conflict

The end customer interacts directly with the Apsis One API using their own email addresses. They want to find and update the same profiles that were synced in from the CRM — but those profiles live in the **CRM/Join CX keyspace**, identified by CRM ID.

The end customer has no knowledge of CRM IDs. They only know email addresses. As a result:

- If they try to **read** data (e.g., fetch loyalty points or events) using an email address, they get **no results** — no profile exists in the email keyspace.
- If they try to **write** data (e.g., add a secondary mobile number) using an email address, the system **creates a duplicate profile** in the email keyspace, separate from the CRM-synced profile.

> "Whatever their use case, it will either create duplicates or return no matching data." — Erik Andersson

This is not a standard setup for the generic connector, but it is a valid use case.

---

## Solution 1: Allow Keyspace Selection at Installation (Full-Fledged)

### Description

Add a configuration option (e.g., a dropdown in the installation UI) allowing customers to specify which keyspace the integration should use — email keyspace instead of the default CRM ID keyspace.

### Problems Identified

This is the most complex solution and has significant caveats:

1. **Unique identifier must also change**: The email keyspace requires a valid email address as the profile key — not a CRM ID string. The integration code currently extracts CRM IDs from contacts; it would need to extract email addresses instead.
2. **Consent updates**: For consent updates, the CRM currently sends only CRM IDs (no email attribute). These would need to be changed to send email addresses as the record ID.
3. **Requires work on both sides**: Apsis engineering must build the keyspace selection feature; Intermail/Join CX must change what identifier they send.
4. **Mutually exclusive with multiple profiles per email**: If email is the keyspace key, one email = one profile. Multiple profiles per email address cannot coexist. Erik confirmed this is a known trade-off.
5. **Scope**: The customer wants this for **one specific installation**, not all Join CX customers — making a general solution even harder to justify.

### Verdict

Not recommended for implementation now. Estimated effort would be significant on both the Apsis side and the CRM side.

---

## Solution 2: Post-Sync Merge into Email Keyspace

### Description

Keep the existing sync logic unchanged (CRM ID → Join CX keyspace). After each profile is created or updated, send an additional **merge request** to the integration profile merge worker service, merging the CRM-keyspace profile with the email keyspace using the email attribute from the contact.

### How the Merge Worker Already Functions

The merge worker already exists in the integration layer. It accepts:
- A **source key** and **source keyspace** (already available: CRM ID + Join CX keyspace)
- A **destination key** and **destination keyspace** (would be: email address + email keyspace)

Current primary use cases for the merge worker:
- **Form submissions**: when a form is submitted, a profile exists in the form/email keyspace; the CRM responds with a new entity ID (e.g., a lead); a merge is triggered to link that form-submitted profile to the CRM-keyspace profile.
- **CRM-initiated merges**: the CRM can notify Apsis that two contacts are actually the same (e.g., duplicate contacts, or a lead promoted to a contact).

### Implementation for This Use Case

1. After each profile update/create from Join CX, extract the email attribute from the contact.
2. If an email exists, publish a message to the integration profile merge worker service with:
   - Source: CRM ID + Join CX keyspace
   - Destination: email address + email keyspace
3. If no email exists, skip the merge step.
4. If the destination profile doesn't exist yet in the email keyspace, the merge worker creates an empty profile — this is acceptable behavior.

> "You don't need to [have a pre-existing profile]. If it doesn't exist, I think they create an empty profile. So that's not a problem." — Erik Andersson

### First-Iteration Shortcut

For this one customer, the implementation could be **hardcoded** (e.g., `if customer_id == X, trigger merge`). The email keyspace ID is stable and could also be hardcoded. This avoids building a full UI/configuration layer.

Estimated effort: **3–5 days**.

### Verdict

Erik's preferred solution if any Apsis-side code change must be made. All the work falls on the Apsis side. The customer and Intermail don't need to change anything.

---

## Solution 3: Database-Level Keyspace Swap (Workaround/Hack)

### Description

At installation time, the integration stores a **mapping table** in the database that maps entity types (e.g., "member", "contact") to keyspace IDs. This is how the integration knows which keyspace to use when it receives an update for a given entity.

The workaround: **manually change the keyspace ID in this mapping table** for the specific customer installation — swapping the Join CX keyspace for the email keyspace. No code changes required on the Apsis side.

### What This Achieves

Once the keyspace is swapped in the database, all profile updates routed through the integration will automatically target the email keyspace instead of the CRM keyspace. The end customer can then find profiles via email address as expected.

### Caveat: Identifier Must Also Change

The integration code will still look for a **CRM ID** as the unique identifier. However, the email keyspace requires a valid email address as the key.

**Therefore**: Intermail must be asked to send the **email address value in the CRM ID field** for all updates — profile updates, consent updates, etc.

> "We will still need them to send us the email address value in the CRM ID, meaning this solution will be a workaround. It will be very swift on our side — I can do that for them this afternoon. But this time around it will require work from them." — Erik Andersson

### Why Consent Updates Make Full Apsis-Side Handling Impossible

Erik confirmed that doing this purely on the Apsis side (without Intermail's changes) would be impossible for consent updates specifically:

> "For consent updates they don't send us any email attributes, they just give us CRM ID. So we can't fix this for consent. We will rely on them having that change on their side and if we rely on them having that change on their side, there is no point in us doing anything manual on our side — they might as well do it for all of the updates." — Erik Andersson

### Implementation Steps (Solution 3)

1. Coordinate with Intermail: request they send **email address as the value in the CRM ID field** for all updates (profile and consent).
2. Once Intermail confirms the change is deployed, update the keyspace mapping in the database for this installation.
3. Document this hack — record that this non-standard database change was made for this specific customer and integration, and why.
4. Synchronize the timing: the database change should happen **after** Intermail has deployed their side, to avoid a window where CRM IDs are still being sent to the wrong keyspace.

### Risks and Gotchas

- **Reinstallation risk**: If the customer reinstalls the Join CX integration, the installation process will overwrite the database mapping back to the default Join CX keyspace. The manual database change would need to be re-applied. This must be documented as an ongoing operational risk.
- **Existing profiles in CRM keyspace**: Profiles already synced and stored under the Join CX/CRM keyspace will **not** automatically move. After the change, new syncs will go to the email keyspace, creating a split:
  - Old profiles: CRM keyspace
  - New/updated profiles: email keyspace → potential duplicates

  Resolution options:
  - Manual merge work by the customer before switching
  - Deleting all existing profiles and doing a full re-sync
  - ⚠️ It is unclear whether this customer already has existing synced profiles. This should be confirmed before proceeding.

### Verdict

Both Erik and Lukasz agree this is the most pragmatic solution given that it affects only one customer and is an exceptional case. Erik personally favors this option.

> "I really like it. I don't have any issues with this... having in mind this is only for one customer and this is an exceptional case." — Lukasz Grabowski

---

## Solution 4: Do Nothing (Customer Adapts to the System Design)

### Description

Apsis makes no changes. The end customer adjusts their custom solution to work with the CRM ID and the Join CX keyspace — i.e., they use the system as designed.

### Verdict

Unlikely to be accepted by the customer. Mentioned for completeness.

> "The system — this integration — was designed to rely on CRM ID and not email." — Lukasz Grabowski

---

## Next Steps and Follow-Up

- **Internal meeting**: Schedule with **Alexander** and **Opri** (optional) to align internally. Lukasz to set this up for the following day at 3 PM.
- **Customer-facing meeting**: After internal alignment, loop in **Max** (customer contact) to present the proposed approach.
- **Proposal to present**: Lead with Solution 3 (database swap + Intermail sends email in CRM ID field). Do not present Solution 1 externally — keep it for internal reference only.
- **Items to raise in the meeting**:
  - Reinstallation risk
  - Handling of existing profiles (possible duplicates)
  - Confirmation of whether existing synced profiles are already in the system
  - Coordination timeline for Intermail's change vs. the database swap

---

## Key Takeaways

1. **The fundamental conflict**: The Join CX generic connector bootstraps a CRM ID keyspace at install time. The end customer's system only knows email addresses. These two identifiers and keyspaces cannot be used interchangeably without intervention.

2. **Two categories of solutions**: (a) Change Apsis-side logic (Solutions 1 and 2 — more effort, more robust long-term), (b) Change the database mapping and ask Intermail to change what they send (Solution 3 — fast, hacky, but pragmatic for a single customer).

3. **Consent updates are the blocking constraint** for a purely Apsis-side fix: consent payloads carry only CRM ID, no email attribute, so Apsis cannot extract an email to use as the key without Intermail's cooperation.

4. **The merge worker already exists** and supports source/destination keyspace merge operations. Solution 2 leverages this without requiring new infrastructure.

5. **Reinstallation is an ongoing operational risk** for Solution 3 — it must be documented and communicated clearly.

6. **Email keyspace uniqueness is non-negotiable**: one email = one profile. Multiple profiles per email address are incompatible with using email as a keyspace key.

7. **This is a non-standard, one-off setup**. Any solution chosen should be documented as exceptional, not as a template for future installations.

---

## Unresolved Questions

- Does this specific customer already have existing synced profiles in the Join CX keyspace? (Affects migration complexity for Solution 3.)
- Is there appetite from product to build Solution 2 (post-sync merge) as a general feature? Alexander/Opri may have insight into whether other customers have similar needs.
- What is Intermail's timeline for changing the CRM ID field to carry email addresses?
- Who is responsible for re-applying the database change if the customer reinstalls?