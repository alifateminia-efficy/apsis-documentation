---
source_file: Erik from Dec 11th 2025 (continued).txt
domain: Apsis One Integrations
topics: [Consent Message Processing, Magento Consent Mapping, Auto-Mapping Logic, Message Processor Error Handling, Consent Changes in External CRMs]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Message Processor, Consent Mappings Client, Magento Integration, Tribe Integration, Event Listeners, Consent Mapping Database]
session_type: debugging-session
subdomains: [Architecture, Different Types of Connectors, Magento]
---

## Session Overview

Erik Andersson walked through a critical debugging discovery involving consent message processing failures in the Magento integration. The root cause centers on how **Magento handles consent mapping differently** from other integrated CRMs—consent mappings are configured inside Magento rather than in the Apsis integration layer. This architectural difference, combined with the **auto-mapping feature introduced in Consent 2.0**, creates a logic bug where consent updates with both email and SMS channels set to true fail to extract the source entity. The team identified that this issue hasn't surfaced in production because no active Magento customers are currently using consent features, but the fix is necessary for future robustness.

---

## Magento Consent Mapping Architecture vs. Standard Integration Pattern

### How Most Integrations Handle Consent Mapping

[Erik Andersson]: For most integrations we work with—like Tribe—consent and subscription mappings are configured on the integration page itself. For example, in Tribe you would set up a mapping where a product like "home" maps to the email channel for the default subscription.

### Magento's Different Approach

[Erik Andersson]: Magento is different because it handles consent mapping **inside Magento itself**, not within the Apsis integration configuration. This means the mappings don't exist on our side, and we don't have visibility into what subscriptions the customer is actually interested in.

> "For such integrations that handle the consents inside of the CRM system, there is a property called **auto mapping**. And this is only for Magento that this is happening right now."

This architectural difference is the root cause of the downstream issues in message processing.

---

## Consent 2.0 and Auto-Mapping Implementation

### Event Listener Registration During Installation

[Erik Andersson]: With Consent 2.0, when **automatic consent mappings is enabled** (set to `true` on the installer option), the integration manager registers event listeners for all subscriptions during installation. However, we **do not set up any consent mappings on our side** in the database.

Instead, we listen to changes on the **assignment attributes for the channels** (email and SMS). When anyone opts in or opts out on these channels for any subscription, we receive that event inside the integration.

### Virtual Consent Mapping Generation

[Erik Andersson]: To make the flow actually work, we generate **virtual consent mappings** at runtime. Here's what happens in the `consent mappings` function:

1. We check if the integration uses `auto_maps: true`
2. If so, we export everything from the audience
3. We return virtual mappings in the required format

For these virtual mappings, we set:
- **source entity**: `contact`
- **topic ID**: extracted from subscription
- **update_email**: `true`
- **update_sms**: `true`

> "We set update_email to true and update_sms to true because we don't know if they are interested in email or SMS or both."

This is the critical setup for the bug that follows.

---

## The Message Processor Bug: Double Condition Failure

### Where the Error Occurs

[Erik Andersson]: The error is thrown in the **message processor at line 426**. The function returns an error if the **source entity is an empty string**, which indicates no consent mapping was found that matches the topic and channel combination.

### The Flawed Logic Flow

The message processor iterates through consent mappings and checks:

1. Does this mapping have a topic ID that matches the subscription?
2. If it's an **email channel** update, is `update_email` set to true and `update_sms` set to false?
3. If both conditions pass, set the source entity

Then it checks the inverse:

4. If it's an **SMS channel** update, is `update_sms` set to true and `update_email` set to false?
5. If both conditions pass, set the source entity

### The Bug: What Happens with Both Flags True

[Erik Andersson]: Now here's the problem. When we have a consent mapping with **both `update_email: true` AND `update_sms: true`** (which is what we generate for Magento auto-mapping):

- We check: "Is this email channel AND update_email true AND update_sms false?" → **False** (because update_sms is true)
- We check: "Is this SMS channel AND update_sms true AND update_email false?" → **False** (because update_email is true)
- We never set the source entity → Empty string returned → Error thrown → **Consent message skipped**

The conditional requires that the **opposite channel's flag be false**, but in the virtual mapping, both are always true. This creates a logical impossibility.

> "So if we have update_sms and update_email set on the consent mapping, we check 'is it true but also false?' and the answer is no, so we just completely ignore this consent update message."

---

## Why This Bug Hasn't Been Detected

### No Active Magento Customers Using Consent

[Erik Andersson]: We checked with Opera and found that while we have two customers with Magento installations, **they are not actually using it**. It's essentially a dead integration for them.

### Single Mappings Per Channel in Other Integrations

[Erik Andersson]: For every other integration (like Tribe), each consent mapping in the database specifies either email **or** SMS, not both. One single mapping will never have both channels enabled unless it was manually entered incorrectly. Therefore, this logic error cannot occur in any other real-world integration.

> "This would have been a real pickle if this had been applicable for other integrations. Luckily this is not the case."

Because of this architectural difference, the bug only affects Magento's auto-mapping behavior.

---

## The Fix

### Immediate Solution

[Erik Andersson]: The fix is simple: **remove the double condition clause**. Instead of requiring that the opposite channel's flag be false, we should only check:

- "Is this mapping for the email channel AND does it have update_email set?" → Set source entity
- "Is this mapping for the SMS channel AND does it have update_sms set?" → Set source entity

The presence of additional flags doesn't matter—we're only extracting the source entity.

### Implementation Impact

With this fix:
- If a consent update comes for the email channel, we find a mapping with `update_email: true` and extract the source entity (even if `update_sms` is also true)
- The additional flag doesn't prevent entity extraction
- Future CRM systems with similar auto-mapping requirements will work correctly

[Erik Andersson]: "More importantly, if you were to implement a new CRM system in the future that has something similar—where you don't know if they're interested in email and SMS—and you set `auto_mapper: true` with both flags, then we should fix this just to have it properly working."

---

## Outstanding Questions and Next Steps

### Customer Account Cleanup

[Michal Rosikiewicz]: Should we contact those Magento customers to remove this integration from their account?

[Erik Andersson]: We should talk with Opera about it first to understand the customer situation better.

### Story Creation and Ticket Work

[Erik Andersson]: "I will create a ticket to tweak this. I have also kidnapped the pair duty for myself for today, so you don't need to bother with it any more for now."

[Michal Rosikiewicz]: A story was already created but with the previous (incorrect) conclusion. Erik will update it with the correct fix.

### Planned Next Meetings

[Michal Rosikiewicz]: We need another session, possibly after Tuesday (when guests are present), to discuss **QoS and backoff strategy**. The team has been dealing with constant alarms for 6 months due to related consent processing issues.

### Implementation Schedule

Tomasz Kowalski will be unavailable tomorrow and Monday. Erik wants to prepare at least 2 of the 4 planned stories before Tomasz returns, so he has clear work to start with immediately.

[Erik Andersson]: "At least I want to have 2 of the 4 stories so you have something you can start on. I think this is about 4 separate stories that would be needed to get this working and all of them are quite independent."

---

## Key Takeaways

1. **Magento's architecture is fundamentally different**: Consent mappings exist in Magento, not in Apsis, requiring a special **auto-mapping** feature that generates virtual mappings at runtime with both email and SMS flags set to true.

2. **The bug is architectural, not operational**: The message processor's double-condition logic (checking that one channel flag is true while the opposite is false) is incompatible with Magento's virtual mappings, which have both flags set to true.

3. **The bug hasn't surfaced because no active customers use it**: The two Magento customers on the platform don't actually use consent features, making this a latent defect.

4. **The fix is straightforward but important for future robustness**: Removing the opposite-channel check ensures that any CRM system with similar auto-mapping behavior will work correctly.

5. **There are 4 independent stories needed**: This fix is just the first step. Additional work is planned around consent handling, auto-mapping cleanup, QoS, and backoff strategies.

6. **Hands-on implementation is crucial for learning**: While walkthroughs provide context, actually fixing the bugs in the system is where deep understanding develops.

---

## Unresolved Questions and Action Items

- **[ASSIGNED TO: Erik Andersson]** Update the ticket/story with correct fix description (removing double condition clause in message processor)
- **[ASSIGNED TO: Erik Andersson]** Talk with Opera about whether to proactively contact the two Magento customers to remove the integration
- **[ASSIGNED TO: Erik Andersson]** Prepare at least 2 of the 4 planned stories before Tomasz returns (he's out tomorrow and Monday)
- **[PENDING]** Schedule follow-up session to discuss QoS and backoff strategy (constant alarms for 6 months)
- **[PENDING]** Create epic for remaining Tribe-related consent issues (mentioned as "another headache for tomorrow")