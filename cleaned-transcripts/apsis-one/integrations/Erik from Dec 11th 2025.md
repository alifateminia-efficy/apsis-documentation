---
source_file: "Erik from Dec 11th 2025.txt"
domain: Apsis One Integrations
topics: [Consent 1.0 vs Consent 2.0 migration, event listener lifecycle, Magento connector, subscription assignment attributes, stale global email unsubscribe listeners, Tribe connector, K contact field mapping, delegation tokens, installation table]
speakers: ["Erik Andersson (engineer, domain expert)", "Michal Rosikiewicz (engineer, session facilitator)"]
key_components: [Magento connector, Tribe connector, Consent 2.0, subscription assignment attributes, event listeners, delegation tokens, installation table, AVS, file import flow]
session_type: knowledge-transfer
---

## Session Overview

Erik and Michal investigate two separate error conditions surfacing in the Apsis One Integrations system. The first and more fully resolved issue concerns stale **global email unsubscribe event listeners** left over from Consent 1.0 that were never cleaned up during the migration to Consent 2.0, specifically affecting Magento connector accounts. The second issue involves a Tribe connector error related to a **K contact field mapping** that could not be fully diagnosed within the session due to tooling instability (AVS crashes). The session establishes clear remediation steps for the Magento issue and flags the Tribe issue for continued investigation.

---

## Consent 1.0 vs. Consent 2.0: Architecture Difference and Migration Gap

### How Consent Was Handled Before Consent 2.0

In the pre-Consent 2.0 architecture, integrations could not listen to attribute change events. The only mechanism available for consent handling was listening to **global email unsubscribe events**.

- When any connector that handled CRM consent was installed, a **single global email unsubscribe event listener** was registered on the account.
- This listener covered all subscriptions — it was not per-mapping.
- When an unsubscribe event was received, the integration would determine which subscription was connected to that event and forward an **opt-out request** to the CRM system.
- **Opt-in events could not be handled** at all under this model because no global opt-in events were available.

### How Consent 2.0 Works

Consent 2.0 introduced **subscription assignment attributes** — hidden per-profile attributes tied to a specific topic/subscription. The integration now works as follows:

- When a consent subscription mapping is configured (via the **consent subscription mapping tab** in the connector UI), the integration registers an event listener specifically on the **assignment attribute** for each mapped subscription.
- When the assignment attribute changes, the integration checks the value: `0`, `1`, or `2`, and sends the corresponding opt-in or opt-out message to the CRM system.
- This is handled at the **assignment attribute level**, per mapping, rather than globally.

[Erik Andersson]: *"This event should never even have reached integration anymore."*

### The Migration Gap: Stale Global Unsubscribe Listeners

The root problem: during migration to Consent 2.0, the old global email unsubscribe event listeners were **not removed** from affected accounts. This means:

- Certain Magento connector accounts still have a registered global email unsubscribe listener from the Consent 1.0 era.
- Events continue to arrive via this listener.
- The current integration code loops through consent mappings to handle incoming events, but finds **no mappings** (either they were never configured, or were removed), causing the event handler to fail.
- This manifests as errors in the logs when file imports or form submissions trigger profile attribute changes that propagate to the stale listener.

> "This listener shouldn't exist on their account and that is why it is failing."
— Erik Andersson

**Why this is scoped to Magento:** Magento is not a widely used connector. This issue would appear for every customer if it were a global regression, but it is isolated to accounts where the old listener was registered and never cleaned up.

---

## Affected Accounts and Log Analysis

During the session, Erik and Michal reviewed logs to identify which accounts are affected.

Findings:
- Several accounts showing errors are **internal/personal accounts** (including a CPS — Professional Services team — account). These are not customer-impacting.
- **Two external customer accounts** were identified as requiring remediation: one unnamed account and **Palm Mark**.
- **Mergular** also appears in logs with the same error pattern (`constant message source entity not found`), but upon inspection, that instance was triggered by a **form tool submission** (not a file import), and is adding data rather than removing it. Erik noted this may be a slightly different scenario.
- The error is not continuously recurring — it fires at the time of file imports or specific events, not constantly.

### Error Pattern

The error occurs because:
1. The stale global email unsubscribe listener receives an event.
2. The integration code tries to find a consent mapping for the event's subscription.
3. No mapping exists → error thrown.

[Michal Rosikiewicz]: *"This could be one of the solutions — change it to a warning — but we still would like to get rid of this listener."*

---

## Remediation Plan for Stale Magento Event Listeners

### Option 1: Code-Level Suppression (Short-term)
Change the error to a **warning** in the code when no mapping is found for an incoming event of this type. This would stop the noise immediately without requiring account access.

### Option 2: Delete the Stale Listener (Preferred)

The preferred solution is to **remove the stale global email unsubscribe event listener** from affected accounts. Two paths to do this:

**Path A — SOC-assisted access:** Have SOC provide staff-user access to the customer account, obtain a UI token, and manually unregister the listener via the API.

**Path B — Delegation token (internal API):** Every integration installation already generates a **delegation token** stored in the **installation table**, used for the inbound flow (delta sync, full sync). This token can be used to generate an access token and programmatically remove the listener **without requiring customer involvement**.

[Erik Andersson]: *"We have that for every integration, so in theory we can do that."*

> The delegation token and One API client keys are stored in the **installation table**. The table contains both the One API data and the delegation data needed to generate access tokens.

**Important note on re-registration:** After the listener is removed, reinstalling the integration core will **not** re-create the global listener. The new code registers per-mapping attribute listeners only, not global unsubscribe listeners.

### Customer Communication Considerations

The question was raised about whether to notify customers that the listener is being removed and ask them to configure consent mappings.

Erik's position:
- These customers likely never had consent mappings configured (or removed them at some point). The integration has **never** removed consent mappings during the Consent 2.0 transition.
- Therefore, there has been no actual consent sync happening for them — removing the listener doesn't break anything that was working.
- It is not strictly necessary to contact them, but SOC *could* optionally reach out to ask if they want consent syncing and guide them to set up mappings.

Michal's position:
- Agreed that it's not urgent, but customers should be made aware that if they want subscription change events to flow through to the CRM, they need to configure consent mappings in the connector UI to create the new Consent 2.0 attribute-level listeners.

**Agreed outcome:** Remove the listeners (preferably via delegation token). Optionally have SOC contact the two affected customers to inform them and offer guidance on setting up consent mappings if desired.

---

## Postman Collection Gap: Missing Delete Event Listener Endpoint

During investigation, Michal checked the **Integrations Postman collection** and found it only has a **GET event listeners** call. A **DELETE event listener** call does not exist in the collection.

This would need to be added or executed manually when performing the listener cleanup.

---

## Tribe Connector: K Contact Field Mapping Error (Unresolved)

### Error Description

A separate error was identified for a Tribe connector account (**Hola Press**, account ID referenced as `21641` in logs). The error involves a **K contact field** host attribute.

The error was triggered by a **mail open event**. The investigation involved:
1. Checking the incoming attribute data from Audience (Audience is configured to include email, SMS, CRM ID, and lead ID in all event payloads to integrations).
2. Checking the mappings table: `WHERE section_id = <id>` for Hola Press to identify configured mappings.
3. Attempting to identify what the specific attribute `07/6` maps to — identified as a K contact attribute.
4. The mapping exists in the table, but the error still occurs.

### Hypothesis

The code path fails somewhere when looping through mappings — the `mapping system field` lookup returns nothing in a specific edge case. The exact condition triggering this is unclear.

[Erik Andersson]: *"What is even more weird is that it is a singular error message, because if this had been something for the whole installation then we would have seen more of them."*

The singular nature of the error suggests it may be tied to a specific profile's data state rather than a systematic misconfiguration.

### Investigation Was Incomplete

Erik was unable to complete the investigation because **AVS** (the internal tooling used for log/data queries) was crashing repeatedly during the session.

[Erik Andersson]: *"I need to check more because I don't fully know yet why this is."*

---

## Key Takeaways

1. **Consent 1.0 global email unsubscribe listeners are obsolete** and should not exist on any account. Their presence is a migration artifact, not intentional behavior.
2. **Consent 2.0 handles consent via per-subscription assignment attribute listeners**, registered only when a consent mapping is explicitly configured by the user.
3. **The Magento connector is the identified connector** with stale Consent 1.0 listeners on a small number of accounts. This is not a widespread issue across all integrations.
4. **Delegation tokens stored in the installation table** can be used to perform administrative actions (like removing event listeners) on customer accounts without requiring direct customer involvement or SOC-provided access.
5. **Affected customers without consent mappings are not in a "newly broken" state** — consent sync was never functioning for them under Consent 2.0 either. Cleanup is safe to perform silently.
6. **The Tribe/K contact field error is a separate, unresolved issue** requiring further investigation once AVS tooling is stable.
7. **The Integrations Postman collection is missing a DELETE event listener endpoint** — this is a tooling gap.

---

## Unresolved Questions and Action Items

- [ ] **Investigate Tribe / K contact error further** (Erik) — determine why the mapping lookup fails for a single event on Hola Press account `21641`. Check whether other K contact attributes on this account show similar anomalies in logs.
- [ ] **Identify all affected Magento accounts** from the last month of logs (Michal) — scope the full list of accounts with stale global unsubscribe listeners.
- [ ] **Remove stale event listeners** from the two confirmed external customer accounts (Palm Mark + one other) using delegation tokens from the installation table.
- [ ] **Determine whether to change the "no mapping found" error to a warning** in code as a short-term noise-reduction measure while listener cleanup is in progress.
- [ ] **Optionally have SOC contact affected customers** to inform them about the consent mapping setup if they want subscription change sync to work.
- [ ] **Add DELETE event listener call to the Integrations Postman collection.**
- [ ] ⚠️ **Ambiguity — Mergular account:** It is unclear whether Mergular's error is the same root cause (stale listener + no mappings) or a distinct issue. The source was a form tool submission. Needs separate verification.