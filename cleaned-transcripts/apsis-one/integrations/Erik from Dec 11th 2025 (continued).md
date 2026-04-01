---
source_file: "Erik from Dec 11th 2025 (continued).txt"
domain: Apsis One Integrations
topics: [Magento consent integration, auto-mapping mechanism, consent 2.0, subscription mappings, sub worker message processing, bug root cause analysis]
speakers: ["Erik Andersson (senior developer/domain expert)", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [sub worker, message processor, consent mappings client, integration manager, Magento integration, consent 2.0]
session_type: debugging-session
---

# Magento Auto-Mapping Consent Bug — Root Cause Analysis

## Session Overview

Erik Andersson walks Michal Rosikiewicz and Tomasz Kowalski through a bug discovered in the sub worker where Magento consent change messages are being silently dropped. The root cause is a logic flaw in the message processor's consent mapping lookup, which is triggered by a special auto-mapping mechanism unique to the Magento integration. Erik identifies the exact condition causing the failure, explains why it has not caused a customer-facing incident yet, and outlines the fix. The session also touches on follow-up work items and scheduling.

---

## Background: How Consent Mappings Work for Standard Integrations

For most integrations, **consent mappings** (also called subscription mappings) are configured manually on the integration page. For example, in a standard integration like Tribe, you would map a product (e.g., "Products Home") to the email channel for the default subscription.

After **Consent 2.0**, any subscription that has a mapping configured will cause the system to start listening for changes on the **assignment attributes** for those channels. So if anyone opts in or out on the email channel for the default subscription, the integration receives that event.

---

## Why Magento Uses Auto-Mapping Instead of Manual Consent Mappings

Magento is a special case: **consent mapping is handled inside Magento itself**, not inside the Apsis One integrations configuration UI. Because of this, the integration team does not set up consent mappings on their side for Magento.

To accommodate integrations where the CRM system manages its own consent mapping externally, there is a property called **`automatic_consent_mappings`** (also referred to as `auto_mapping` or `auto_maps`).

This is currently **only used by the Magento integration**.

In the integration installer options, this is configured as:

```
automatic_consent_mappings: true
```

During installation, in the **Integration Manager**, if `automatic_consent_mappings` is enabled, the system registers event listeners for everything in the subscription section — but **does not set up any consent mappings on the Apsis side**.

---

## How Auto-Mapping Generates Virtual Consent Mappings at Runtime

When the system needs to retrieve consent mappings (via the **consent mappings client**), there is a special function clause that handles the auto-mapping case:

> "If the integration is auto-maps — that is, it is using these external consent mappings — then we set up like virtual consent mappings just so our flow will actually work."

To generate these virtual consent mappings, the system exports everything inside the audience and returns a map in the required format. Critically, the virtual mapping is constructed with:

- `source_entity` = `contact`
- `topic_id` = set appropriately
- `update_email` = `true`
- `update_sms` = `true`

**⚠️ Important — foreshadowing the bug:** Both `update_email` and `update_sms` are set to `true` on every virtual consent mapping, because the system does not know in advance whether the subscriber is interested in email, SMS, or both.

For all other (non-Magento) integrations, each manually configured consent mapping row in the database specifies either email **or** SMS — never both.

---

## The Bug: Message Processor Logic Fails When Both `update_email` and `update_sms` Are True

### Location

The failure occurs in the **message processor at line 426**, which logs:

> "We are skipping the consent message because we could not find any corresponding source entity for the consent."

### Root Cause

The message processor loops through all retrieved consent mappings to find a match for the incoming consent change event. The matching logic works as follows:

1. Check if the mapping has a `topic_id` and it matches the subscription the consent change is for.
2. If the incoming event is for the **email channel**: check that the mapping has `update_email = true` **AND** `update_sms = false`.
3. If the incoming event is for the **SMS channel**: check that the mapping has `update_sms = true` **AND** `update_email = false`.
4. If either condition is met, set the `source_entity` and proceed.

The bug: for Magento's virtual consent mappings, **both `update_email` and `update_sms` are `true`**. This means:

- When checking for an email channel event: `update_email = true` ✓ but `update_sms = false` ✗ → condition fails
- When checking for an SMS channel event: `update_sms = true` ✓ but `update_email = false` ✗ → condition fails

Neither branch matches, so `source_entity` is never set, the mapping is effectively invisible to the processor, and the consent message is silently dropped.

[Erik Andersson]: *"I'm going to give you 30 seconds to try and find what happens here if we have `update_sms` and `update_email` set on the consent mapping."*

---

## Why This Has Not Caused a Customer Incident

[Erik Andersson]: *"I checked with Oprea, and while we have two customers with a Magento installation, they are not actually using it. It's essentially a dead integration for them."*

Because no active customers are actually using the Magento consent flow, there are:
- No live customer-facing incidents
- No data cleanup required
- No need for an incident process

---

## The Fix

The fix is straightforward: **remove the exclusive `update_sms = false` guard clause** from the email channel check (and equivalently for the SMS check).

The corrected logic should be:
- If the consent change is for the **email channel**, find a mapping with the matching `topic_id` that has `update_email = true`. The value of `update_sms` should not matter.
- If the consent change is for the **SMS channel**, find a mapping with the matching `topic_id` that has `update_sms = true`. The value of `update_email` should not matter.

The goal is to extract the `source_entity` — not to gate on the exclusivity of the channel flags.

[Erik Andersson]: *"It would have been a real pickle if this had been applicable for other integrations. Luckily this is not the case."*

---

## Why the Fix Is Safe for All Other Integrations

For every non-Magento integration, each consent mapping row in the database is configured with **either** `update_email = true` or `update_sms = true`, but never both. This is enforced by the UI setup flow. Therefore, removing the exclusivity guard does not change behavior for any existing integration.

This fix also future-proofs the system: if a new CRM integration is built that similarly does not know upfront whether to handle email or SMS consent, it could use the same auto-mapping mechanism and it would work correctly after this fix.

---

## The Unsubscribe Event — Clarification

[Erik Andersson]: *"The unsubscribe one was a red herring. We are actually using the consent structure for it. The unsubscribe was the source of the consent change, but it is not actually the unsubscribe event we reacted on — it was the outcome of the unsubscribe."*

⚠️ [Ambiguous] — this point was stated briefly without full elaboration. The distinction between the unsubscribe event trigger and the downstream consent change outcome may be worth clarifying in the ticket or a follow-up session.

---

## Follow-Up Actions and Story Updates

- **Michal** had already created a story for this bug, but based on the previous (incorrect) conclusion. Erik will update it with the correct root cause and fix.
- **Erik** will create a ticket for the fix.
- **Erik** has also taken on-call/paid duty for today and the next day to allow the team to focus.
- **Question raised by Michal**: Should Customer Success (or "Suck" — likely an internal team name) contact the two Magento customers to discuss removing the integration from their accounts? Erik will follow up with Oprea on this.

---

## Upcoming Work: SQS and Back-off Strategy (Noted, Not Discussed)

Michal raised that there is a separate ongoing issue related to **SQS and a back-off strategy** that has been generating constant alarms for approximately 6 months. This was explicitly deferred to a future session (possibly the following Tuesday), as it requires changes to the processing logic and there are currently guests present.

[Michal Rosikiewicz]: *"We have constant alarms for like 6 months and we cannot do much about it without changing the logic a bit."*

---

## Key Takeaways

1. **Magento is the only integration using `automatic_consent_mappings: true`**, a mechanism that generates virtual consent mappings at runtime with both `update_email` and `update_sms` set to `true`.
2. **The message processor's consent mapping lookup has an overly strict exclusivity check** that assumes no single mapping will have both `update_email` and `update_sms` as `true` — an assumption that is violated by the Magento auto-mapping design.
3. **The fix is to drop the exclusivity guard** (i.e., stop requiring `update_sms = false` when matching on email channel, and vice versa).
4. **No customer impact has occurred** because both existing Magento customers are not actively using the integration.
5. **This bug class could recur** if any future CRM integration uses the auto-mapping mechanism without the fix in place — the fix is therefore important for future-proofing.
6. For all manually configured integrations, each consent mapping row specifies exactly one channel (email XOR SMS), so the existing behavior for those integrations is unaffected by the fix.

---

## Unresolved Questions and Action Items

| Item | Owner | Status |
|---|---|---|
| Update the existing story with the correct root cause and fix details | Erik Andersson | In progress |
| Determine whether to contact the two Magento customers about removing the integration | Erik Andersson (to consult Oprea) | Pending |
| Clarify the exact distinction between the unsubscribe event and the consent change outcome for Magento | Erik Andersson | Not yet addressed |
| SQS / back-off strategy discussion | Full team | Deferred to future session (post Tomasz's return) |
| Create epic and ~4 implementation stories for the broader fix work | Erik Andersson | In progress; Tomasz is the primary implementer and will be away Thu–Mon |