---
source_file: "Keyspaces in Integrations part 3 (solutions).txt"
domain: Apsis One Integrations
topics: [keyspace configuration, profile identity resolution, CRM ID vs email identifier, profile merge worker, integration workarounds, duplicate profile prevention]
speakers: ["Erik Andersson (integration/platform engineer)", "Lukasz Grabowski (developer)"]
key_components: [Join CX connector, Apsis One, email keyspace, CRM keyspace, profile merge worker service, consent subscription updates, integration installation database]
session_type: knowledge-transfer
---

## Session Overview

Erik and Lukasz work through a specific customer integration problem involving a mismatch between how the Join CX generic connector identifies profiles (CRM ID / Join CX keyspace) and how the end customer wants to interact with Apsis One (email address / email keyspace). Three realistic solutions and one non-solution are evaluated in detail, ranging from full platform changes to a targeted database-level workaround. The session closes with a plan to loop in internal stakeholders (Alexander, Opri) before engaging the customer (GolfAmore / Intermail).

---

## Problem Definition: CRM ID vs. Email Keyspace Identity Mismatch

### The Customer Setup

The end customer is **GolfAmore**, integrated with a loyalty system called **Join CX**, developed by **Intermail**. The data flow is:

1. End customer (GolfAmore) interacts with Join CX using **email addresses** as their identifier.
2. Join CX syncs data to Apsis One using a **CRM ID** identifier — this is standard behavior for the generic Join CX connector.
3. When the Join CX connector installs, it bootstraps a dedicated **Join CX keyspace** tied to the CRM ID.
4. GolfAmore also directly calls the **Apsis One API** in their own custom solution, but they only know email addresses — they have no awareness of CRM IDs.

### The Core Problem

There is no identity mapping between the two keyspaces:

- Profiles created by Join CX exist in the **Join CX keyspace**, keyed by CRM ID.
- GolfAmore's direct API calls use the **email keyspace**, keyed by email address.
- These are treated as separate, unrelated profiles in Apsis One.

**Consequences of this mismatch:**
- If GolfAmore tries to read data (e.g., loyalty points, custom events) for a profile using an email address, they get **no results** — the profile doesn't exist in the email keyspace.
- If GolfAmore tries to write data (e.g., add a secondary mobile number) using an email address, Apsis One **creates a duplicate profile** in the email keyspace instead of updating the existing CRM keyspace profile.
- Every interaction will either return empty data or produce duplicates.

> [Erik]: "Whatever their use case, it will either create duplicates or return no matching data."

---

## Solution 1: Allow Per-Installation Keyspace Configuration (Full Platform Change)

### Description

Add a UI option (e.g., a dropdown) during the Join CX installation process allowing the installer to select which keyspace the integration should use. For this specific customer, they would select the **email keyspace** instead of the default Join CX/CRM keyspace.

### Why This Is Complex

Beyond the UI change, the **unique identifier logic** would also need to change:

- The CRM keyspace accepts any string as identifier.
- The **email keyspace requires a valid email address** — the connector currently looks for a CRM ID on the profile; it would need to instead extract the **email address attribute**.
- This means changing what attribute is read from contacts during sync (email address instead of CRM ID).
- For **consent subscription updates**, the `record_id` field currently carries the CRM ID — this would need to carry the email address instead, requiring changes on Intermail's side too.

### Constraint: Mutual Exclusivity

> [Erik]: "If they want to use the email keyspace, they cannot support multiple profiles with the same email address. These are mutually exclusive."

This is expected behavior for a keyed system but must be clearly communicated to the customer.

### Assessment

Both Apsis (connector logic changes) and Intermail (payload changes) would need to do significant work. Erik is **not recommending this approach** but listed it for completeness.

---

## Solution 2: Post-Sync Profile Merge into Email Keyspace (Targeted Code Change)

### Description

Keep the existing sync logic unchanged (CRM ID → Join CX keyspace). After each profile create/update, send an additional **merge request** to the **integration profile merge worker service**, merging the newly synced profile into the email keyspace using the profile's email attribute.

This means: after a contact is synced from Join CX into the Join CX keyspace with its CRM ID, a merge message is dispatched that links that profile to the email keyspace entry for the same email address.

### How the Merge Worker Currently Works

The **profile merge worker service** is already used in the integration layer. Its inputs are:

- Source key + source keyspace (e.g., CRM ID in Join CX keyspace)
- Destination key + destination keyspace (e.g., email address in email keyspace)

> [Erik]: "If the destination profile doesn't exist, it creates an empty profile. So that's not a problem."

**Current primary use case:** Form submission handling. When a form is submitted, a profile exists in the form/email keyspace. The CRM responds with a new entity ID (e.g., a Lead ID). A merge operation then links the form-submitted profile to the CRM keyspace profile, preventing duplicates on future CRM-driven updates.

The CRM can also **send inbound merge requests** (e.g., "we accidentally created a duplicate" or "this lead was promoted to a contact, merge them").

### Implementation Notes

- First iteration can be **customer-specific**: wrap in a conditional check (`if customer_id == X`), hard-code the email keyspace ID.
- The email keyspace ID is stable and won't change, so hard-coding is acceptable as a temporary measure.
- Libraries/code for keyspace ID lookup already exist in the codebase.
- A **full sync** would be required after implementation to process all existing contacts and retroactively create the merge links.

### Estimated Effort

[Erik]: 3–5 days  
[Lukasz]: prefers to say 3–5 days as well

### Caveats

- Still requires Intermail to send email addresses in their payloads — specifically for **consent updates**, which currently only contain CRM ID and no email attribute. This means the Apsis side alone cannot fully resolve the problem.

> [Erik]: "For consent updates, they don't send us any email attributes, they just give us CRM ID. So we can't fix this for consent [on our side alone]."

- A full sync is needed to handle **pre-existing profiles** that were synced into the Join CX keyspace before merge logic existed.

### Assessment

Erik's preferred option **if any Apsis-side code changes are required**. More robust than the database hack (Solution 3) and avoids redesigning the integration architecture.

---

## Solution 3: Direct Database Keyspace Remapping (Immediate Workaround)

### Description

The integration stores a **mapping table** in the database that resolves: *for this installation, when I receive an update for entity type X, which keyspace should I use?*

Each installation row contains a **keyspace discriminator** per entity. For Join CX, this currently maps the `member` entity to the Join CX keyspace.

**The workaround:** Directly update this database mapping for GolfAmore's installation, replacing the Join CX keyspace ID with the **email keyspace ID**. No code changes required on the Apsis side.

```
# Conceptual DB change
UPDATE integration_keyspace_mapping
SET keyspace_id = <email_keyspace_id>
WHERE installation_id = <golf_amore_installation_id>
  AND entity_type = 'member';
```

### What This Fixes

After the change, any profile update coming from Join CX for this installation will be applied to the email keyspace instead of the Join CX keyspace. From Apsis's perspective, the integration is using the email keyspace natively — no merge step needed.

> [Erik]: "This will take you 5 seconds to change and then everything in Apsis is fixed essentially."

### What This Does NOT Fix (and the Required Tradeoff)

The connector will still pass whatever value it receives as the **CRM ID field** as the profile key — but the email keyspace requires a valid email address as the key. Therefore:

**Intermail must be changed to send the email address value in the CRM ID field** for all events:
- Profile create/update syncs
- Consent subscription updates (record ID field must be the email address, not CRM ID)

> [Erik]: "This solution will be a workaround. It will be very, very swift [on the Apsis side]. I can do that for them this afternoon. But this time around it will require work from them."

### Risks and Gotchas

1. **Reinstallation risk:** If the customer ever reinstalls the Join CX integration, the installation bootstrap process will overwrite the database mapping back to the default Join CX keyspace. This manual DB change would need to be **re-applied after any reinstallation**.

   > [Erik]: "Typically I don't think that they are doing this like install, reinstall all the time." — but it must be documented.

2. **Existing duplicate profiles:** Profiles already synced into the Join CX keyspace will remain there. After the switch, new/updated profiles will go into the email keyspace. This means the **same contact could exist in both keyspaces** as duplicates until manual merge work is done.

   Options for handling existing profiles:
   - Manual merge work by the customer
   - Delete all existing profiles and re-sync from scratch

   > [Erik]: "How they solve that, that's not really our issue."

3. **This hack must be documented** — in Confluence or equivalent — noting which installation was modified, why, and what the risks are (especially the reinstallation risk).

### Assessment

Both speakers agree this is the **best pragmatic solution** given the scope (single customer, exceptional case). Erik is a strong proponent. The key dependency is Intermail completing their payload changes before the DB switch is made, so the two changes should be coordinated and switched simultaneously.

> [Lukasz]: "Having in mind that this is only for one customer and this is an exceptional case, I really like it."

---

## Solution 4: Do Nothing — Customer Uses CRM ID and Join CX Keyspace (Non-Solution)

The customer adapts their custom integration to use the CRM ID and Join CX keyspace instead of email addresses. This is how the integration was designed to work.

> [Erik]: "I don't think they are going to go for this, but that is of course an easy way out for us."

Both speakers agree this should be listed as an option and communicated to the customer, but it is unlikely to be accepted.

---

## Architecture Note: How Join CX Keyspace Bootstrapping Works

When a Join CX integration is installed:

1. A **dedicated keyspace** is created/registered for each entity type (e.g., `member`).
2. A **keyspace mapping table** is populated in the integration database, associating each entity type with its keyspace ID for that specific installation.
3. All subsequent profile operations for that installation use this mapping to determine which keyspace to read/write.

This is a **per-installation** configuration, not global — meaning a change to one installation's mapping does not affect other Join CX customers.

---

## Next Steps and Follow-Up Plan

1. **Do not present Solution 1** to stakeholders — keep it as internal background only.
2. **Propose Solution 3** (DB workaround) as the recommended path forward, with Solution 2 as the longer-term proper fix if other customers request the same behavior.
3. Schedule a meeting with **Alexander** (and optionally **Opri**, at Alexander's discretion) to align internally before engaging the customer (GolfAmore / Intermail / **Max**).
   - Suggested time: next day at 15:00, 1-hour slot
   - Meeting name reference: "GolfAmore" or "Intermail customer keyspace issue"
4. Lukasz to document the solutions (including this session's notes) on the relevant Confluence page.
5. Coordinate timing with Intermail: Apsis DB change and Intermail payload change should be synchronized. Intermail should notify Apsis when their side is ready.

---

## Key Takeaways

- The root cause is an **identity model mismatch**: Join CX connector uses CRM ID + Join CX keyspace; the end customer uses email address + email keyspace. Apsis One has no automatic translation between these.
- The **quickest resolution** (Solution 3) requires zero Apsis code changes but requires Intermail to send email addresses wherever CRM IDs are currently sent — including consent update `record_id` fields.
- The **merge worker service** (Solution 2) is already functional and used for form submission flows; it could be extended to handle this use case with a customer-specific conditional and 3–5 days of work.
- A **full sync** would be required after implementing Solution 2 to retroactively apply merge links to pre-existing profiles.
- **Reinstallation is a known risk** for Solution 3 — it will revert the DB change, and this must be documented clearly.
- The email keyspace enforces **unique email addresses** — having multiple profiles under the same email is not supported and is mutually exclusive with using email as a keyspace key.
- This is a **one-off exceptional case** for a single customer; neither speaker recommends building a full generalized solution at this time, pending product prioritization discussions.

---

## Unresolved Questions

- **⚠ Does GolfAmore already have existing synced profiles in the Join CX keyspace?** Erik believes they might not yet be fully live, but was uncertain. This affects whether a duplicate-cleanup step is required before switching. Needs confirmation before proceeding.
- **Are there other customers with the same need?** Alexander / Opri may know. If yes, Solution 2 (merge worker extension) becomes higher priority.
- **Priority timeline:** Erik estimates Solution 2 (proper code change) would be deprioritized for 3–4 months given current workload. This needs product confirmation.
- **Action item:** Erik to share diagram/notes document with Lukasz (document sharing was pending at session end).
- **Action item:** Lukasz to schedule 1-hour meeting with Alexander (and optionally Opri) to discuss and align on recommended approach before customer communication.