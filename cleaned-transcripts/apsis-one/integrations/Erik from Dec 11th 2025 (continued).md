---
source_file: Erik from Dec 11th 2025 (continued).txt
domain: Apsis One Integrations
topics: [Magento Consent Mapping, Auto Mapping Logic, Consent Change Processing, Message Processing Flow, Integration Architecture]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Message Processor, Consent Mappings Client, Magento Integration, Tribe Integration, Sub Worker, Assignment Attributes]
session_type: debugging-session
subdomains: [Architecture, Microsoft Dynamics Integration]
---

## Session Overview

Erik walked through a critical bug in the Magento integration's consent change handling. The issue stems from how Magento (unlike most other integrations) handles consent mappings internally within the CRM rather than through Apsis's integration mappings interface. When processing consent changes, the system was failing to locate corresponding source entities due to a logic flaw in handling "auto mapping" scenarios where both email and SMS channels are marked as updatable. The team identified this as a long-standing but undetected issue because no active Magento customers are actually using the feature. Erik outlined the fix and next steps, including creation of tickets for the code correction.

---

## Magento Consent Mapping: Architecture & Differences from Standard Integrations

### How Standard Integrations Handle Consent Mappings

[Erik Andersson]: Most integrations handle consent by setting up explicit mappings on the integration page. For example, in Tribe, you would map subscription products (like "home") to email channels for the default subscription. Each mapping is discrete and well-defined.

### How Magento Handles Consent (Unique Approach)

[Erik Andersson]: Magento is fundamentally different. It handles consent mapping **inside Magento itself**, not within the Apsis integration interface. This means:

- No explicit consent mappings are set up in Apsis
- The CRM system owns the mapping logic
- After consent 2.0 implementation, Apsis listens to changes on assignment attributes for channels (e.g., email channel opt-in/opt-out events for default subscription)
- The integration receives the event but doesn't know which subscriptions the customer is actually interested in

### The Auto Mapping Solution

Because Magento handles mappings externally, Apsis implements a workaround called **auto mapping**. This is specific to Magento integration:

On the installer options, there's a property:
```
automatic_consent_mappings: true
```

[Erik Andersson]: During installation in the integration manager, if `automatic_consent_mappings` is enabled, the system registers event listeners for all subscriptions without setting up explicit consent mappings on the Apsis side. Instead, it generates **virtual consent mappings** on-the-fly by exporting everything from the audience.

---

## The Bug: Logic Flaw in Consent Mapping Retrieval

### Where the Error Occurs

[Erik Andersson]: The error happens in the message processor at line 426. The system returns an error when the source entity is an empty string, meaning no consent mapping could be found that matches the topic (subscription) and channel.

### The Consent Mapping Retrieval Flow

When the consent mappings client retrieves mappings via `get_mappings()`:

1. If the integration uses auto mapping (i.e., external consent mappings), the system generates virtual consent mappings
2. These virtual mappings are created by exporting audience data with required format
3. **Critical detail**: For auto-mapped integrations, the virtual mappings set both `update_email: true` and `update_sms: true` because the system doesn't know which channels the customer opted into

[Erik Andersson]: This is foreshadowing. The system sets both flags to true because we don't know if they're interested in email, SMS, or both.

### The Logic Failure

In the main processing flow, the system iterates through consent mappings checking:

1. Does this mapping have a topic ID that matches the subscription?
2. If it's the email channel, does the mapping have `update_email: true` and NOT `update_sms: true`?
3. If it's the SMS channel, does the mapping have `update_sms: true` and NOT `update_email: true`?

[Erik Andersson]: The problem is when a virtual consent mapping has **both `update_email: true` AND `update_sms: true`** simultaneously. The logic then checks:
- "Is this for email AND only email?" → No (it also has SMS)
- "Is this for SMS AND only SMS?" → No (it also has email)
- Result: Neither condition is satisfied, so the source entity is never set, and the consent update is skipped as if no mapping exists

---

## Why This Bug Wasn't Caught

### Magento Customer Status

[Erik Andersson]: While Apsis has two customers with Magento installations, **neither is actually using the consent feature**. The integration is essentially dormant for those accounts. Therefore, this code path has never been executed in production.

> "I checked with Opera and like while we have two customers with a Magento installation, they are not actually using it. It's essentially a dead integration for them."

### Why Other Integrations Don't Have This Issue

[Erik Andersson]: For every other integration (Tribe, Efficy, etc.), consent mappings are set up explicitly in the database. Each mapping specifies whether it's for email OR SMS, never both in a single row:

> "This will be like one row in the database. Because for every mapping you set up, you specify if it is email or SMS, we will never run into a situation where it is true for both of them. Like one single mapping will not have both email and SMS unless like we had manually entered it."

The Magento auto-mapping logic is the **only place** where a single virtual mapping can have both flags set to true.

---

## The Fix

### Simple Code Change

Remove the redundant conditional logic. Instead of:
```
if (mapping.update_email && topic == EMAIL_CHANNEL && !mapping.update_sms)
   set source_entity
else if (mapping.update_sms && topic == SMS_CHANNEL && !mapping.update_email)
   set source_entity
```

Change to:
```
if (mapping.update_email && topic == EMAIL_CHANNEL)
   set source_entity
if (mapping.update_sms && topic == SMS_CHANNEL)
   set source_entity
```

[Erik Andersson]: The AND check on the opposite channel is unnecessary. If we're looking for an email update, we should match any mapping with `update_email: true` and the correct topic, regardless of whether `update_sms` is also true.

### Why This Still Works

Even with this fix, if someone in the future sets up a Magento customer properly, the logic will work. The mapping is no longer rejected on the false premise that it's "too ambiguous."

---

## Impact Assessment & Future Considerations

### Current Impact

- **No production incidents** because no Magento customers actively use consent features
- No cleanup needed for existing customer data
- Can be deployed whenever convenient

### Future-Proofing

[Erik Andersson]: This fix is important not just for Magento but for any future CRM integrations with similar architecture where:
- The CRM owns the mapping logic internally
- Auto mapping is used
- A single external mapping could apply to multiple channels

> "Even though it is technically not a problem for any other real CRM systems... if you were to implement a new CRM system in the future that has like something similar that you don't know if they are interested in the consent and email and SMS... that's why we actually should fix this just to have it properly working."

---

## Debugging Note: The "Unsubscribe Red Herring"

[Erik Andersson]: During investigation, an unsubscribe event appeared to be the source of the consent change, but it was actually a red herring. The system was correctly using the consent structure, but the unsubscribe was simply the **outcome** of the actual consent change event, not the trigger for processing.

---

## Next Steps & Action Items

### Tickets to Create

[Michal Rosikiewicz]: A story was created for this issue. It needs to be updated with the correct conclusion.

[Erik Andersson]: The ticket should address:
1. Fix the double-condition clause in the consent mapping logic
2. Ensure auto mapping works properly for both email and SMS scenarios
3. Consider whether to contact existing Magento customers about the feature status

### Customer Outreach Decision

[Michal Rosikiewicz]: Asked whether Apsis should contact the two Magento customers to discuss or remove the integration.

[Erik Andersson]: Will discuss with Opera (customer success) before proceeding.

### Related Outstanding Issues

[Michal Rosikiewicz]: There are related issues with QS (Quality/Service?) and backoff strategy with "constant alarms on for like 6 months" that cannot be addressed without changing logic. This is separate from the current fix and will be discussed in a future session.

### Implementation Plan

[Erik Andersson]: This is one of approximately **4 separate, independent stories** needed to fully address consent handling. He plans to prepare at least 2 stories for Tomasz to work on when Tomasz returns (currently out Dec 12 and Dec 15). Having foundational stories ready will allow Tomasz to start implementation and learn by doing rather than just through walkthroughs.

> "I think that's when you will really learn about it and how it is all connected. It is one thing sitting and listening to my boring walkthroughs, but it is a complete other thing, like actually having to fix things in the system."

---

## Key Takeaways

1. **Architecture Mismatch**: Magento's external consent mapping model fundamentally differs from standard Apsis integrations, requiring special handling via auto mapping.

2. **Virtual Mapping Problem**: The auto mapping feature creates virtual mappings with both `update_email` and `update_sms` set to true, which the current logic cannot handle.

3. **Logic Bug**: The consent mapping retrieval code uses AND conditions that reject mappings having both flags set, even though one flag is relevant and the other is irrelevant for any given channel update.

4. **Low-Risk Fix**: The bug has zero production impact because no customers actively use Magento consent features. The fix is straightforward: remove the negation checks on the opposite channel flag.

5. **Broader Relevance**: While Magento-specific today, this pattern may appear in future CRM integrations, so the fix improves the codebase for extensibility.

6. **Hands-On Learning**: The team recognizes that reading walkthroughs is insufficient; implementation work is where real understanding develops.

---

## Unresolved Questions & Decisions Pending

- **Customer Communication**: Should Opera (customer success) contact the two Magento customers about their non-functional consent integrations? (Erik to discuss with Opera)
- **Full Scope**: The 4-story work package for consent handling has broader implications beyond this single bug; full scope to be discussed in future sessions
- **QS & Backoff Strategy**: Separate but related issue with persistent alarms; deferred for future discussion (noted as lower priority than current fix)