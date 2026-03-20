---
source_file: Erik from Dec 11th 2025 (continued).txt
domain: Apsis One - Integrations
topics: [Consent mapping, Magento integration, Auto-mapping configuration, Message processor logic, Consent change event handling]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Message processor, Consent mappings client, Integration manager, Audience configuration, Email/SMS channels, Subscription attributes]
session_type: debugging-session
---

## Session Overview

Erik walked through a critical bug in consent handling for Magento integrations discovered in the old sub worker. The issue stems from a mismatch between how Magento manages consent mappings (internally within Magento) versus how most other integrations handle them (via integration configuration). The team identified a logic flaw in the message processor that fails to extract the source entity when a consent mapping has both `update_email` and `update_sms` flags set to true simultaneously—a condition that only occurs with Magento's auto-mapping feature and has gone undetected because no active customers currently use Magento. The fix involves removing a restrictive conditional check.

---

## Magento Consent Handling vs. Standard Integration Pattern

### Standard Integration Approach

In most integrations (e.g., Tribe), consent mappings are configured explicitly on the integration page:
- Users set up subscription mappings for specific channels
- Example: map the "products_home" topic to the "e-mail" channel for "default subscription"
- The integration maintains explicit knowledge of which subscriptions map to which channels

### Magento's Unique Architecture

Magento handles consent mappings **internally within the Magento system itself**, not within Apsis integrations:
- No explicit subscription-to-channel mappings are set up in the integration configuration
- This creates a knowledge gap: the integration doesn't know which subscriptions Magento customers are interested in
- After consent 2.0, the system listens to changes on assignment attributes for channels, but without pre-configured mappings, it cannot resolve incoming consent changes to known subscriptions

---

## The Auto-Mapping Feature: Virtual Consent Mappings

### How Auto-Mapping Works

To bridge the consent mapping gap for Magento, an **`auto_mapping`** property was introduced (currently only used for Magento):

```
installer_option: automatic_consent_mappings = true
```

When enabled during integration installation:
- The integration manager registers event listeners for all subscriptions without setting up explicit consent mappings on the integration side
- Instead of static mappings, the system generates **virtual consent mappings** dynamically

### Virtual Mapping Generation

When the consent mappings client is called with `get_mappings()`:
- If the integration has `auto_maps = true`, virtual consent mappings are generated on-the-fly
- The system exports everything in the audience and returns it in the required format
- Each virtual mapping includes:
  - `source_entity: "contact"`
  - `topic_id`: the subscription identifier
  - **Both flags set**: `update_email: true` and `update_sms: true`
  
> The system cannot know whether the customer is interested in email, SMS, or both, so both flags are set to true.

This is critical foreshadowing for the bug that follows.

---

## The Message Processor Logic and the Double-Condition Bug

### Error Condition

When processing consent messages, the message processor checks line 426:

```
if (source_entity == empty_string) {
  return error("Could not find any corresponding source entity for the consent")
  skip_message()
}
```

This error occurs when:
- No consent mapping matches the incoming consent change topic
- No entity (contact/lead/etc.) is found

### How Consent Resolution Works

The processor iterates through all consent mappings and attempts to match based on topic ID and channel:

**For EMAIL channel changes:**
```
if (mapping.topic_id == consent.topic AND mapping.update_email == true AND mapping.update_sms == false) {
  set source_entity = mapping.source_entity
}
```

**For SMS channel changes:**
```
if (mapping.topic_id == consent.topic AND mapping.update_sms == true AND mapping.update_email == false) {
  set source_entity = mapping.source_entity
}
```

### The Bug: What Happens When Both Flags Are True?

When a virtual mapping has **both `update_email = true` AND `update_sms = true`**:
- An email channel change: the first condition checks `update_email == true AND update_sms == false`
  - `update_email` is true ✓
  - `update_sms` is false ✗ (it's actually true)
  - **Condition fails**
- The second condition checks `update_sms == true AND update_email == false`
  - `update_sms` is true ✓
  - `update_email` is false ✗ (it's actually true)
  - **Condition fails**
- Result: **no source entity is set**, the message is skipped, and the consent change is lost

[Erik Andersson]: > "This is a very special case for Magento and the reason this has actually not been an issue is because we have checked with Oprey and while we have two customers with a Magento installation, they are not actually using it. It's essentially a dead integration for them."

---

## Why This Bug Hasn't Been Caught

- Only **2 Magento customers** exist in the system
- **Neither is actively using** the integration
- The bug only manifests with auto-mapping enabled and simultaneous email/SMS subscription setup
- No other integration uses the auto-mapping feature, so they never generate mappings with both flags set to true
- Standard integrations have explicit single-channel mappings configured by administrators

---

## The Fix

The solution is straightforward: **remove the restrictive `AND` condition**

Change from:
```
if (mapping.topic_id == consent.topic AND mapping.update_email == true AND mapping.update_sms == false) {
  set source_entity
}
```

To:
```
if (mapping.topic_id == consent.topic AND mapping.update_email == true) {
  set source_entity
}
```

And similarly for SMS:
```
if (mapping.topic_id == consent.topic AND mapping.update_sms == true) {
  set source_entity
}
```

[Erik Andersson]: > "If we remove this double clause because then if they update the email channel then they will find here 'is there a mapping with this topic that has update_email?' Yes there is. The fact that it also happens to have update_sms doesn't matter because it's the source entity that we are after."

### Why This Is Safe

- For all current non-Magento integrations: each mapping has **exactly one channel** (email OR sms, never both), so the fix has no impact
- For Magento with auto-mapping: the fix allows proper entity resolution regardless of multiple flags
- For future CRM systems with similar auto-mapping needs: the logic will work correctly from the start

---

## Unresolved Questions and Action Items

### Known Issues to Address

1. **Consent Alarms**: The team has had continuous alarms firing for 6 months related to consent processing. [Michal Rosikiewicz] noted this cannot be addressed without changing logic, but deferred to a future discussion (scheduled for Tuesday, pending guest availability).

2. **Magento Customer Communication**: [Michal Rosikiewicz] raised whether support should contact the two inactive Magento customers to remove the integration from their accounts. [Erik Andersson] agreed to discuss with Oprey before taking action.

3. **Unsubscribe Event Source**: During debugging, an "unsubscribe" event was initially suspected as the root cause but turned out to be a red herring. The actual consent change mechanism involves the consent structure outcome, not the unsubscribe event itself.

### Work Items Created

- **Story**: Fix the double-condition logic in message processor consent mapping evaluation
  - Include removal of the `update_email AND NOT update_sms` / `update_sms AND NOT update_email` checks
  - Consider the `auto_mapping = true` property as part of the root cause documentation
  - [Michal Rosikiewicz] already created an initial story; [Erik Andersson] to update with correct findings
  
- **Potential future story**: Remove or refactor the Magento auto-mapping feature entirely (deferred pending customer communication outcome)

### Estimated Work

[Erik Andersson] estimates the fix requires approximately **4 separate stories** across different areas, though they are largely independent. The goal is to have at least 2 stories ready before [Tomasz Kowalski] returns (he is away Dec 12-13), so implementation work can proceed in parallel.

---

## Key Takeaways

1. **Architecture asymmetry risk**: Magento's internal consent mapping creates a knowledge gap that requires workarounds. This pattern should be avoided in future CRM integrations.

2. **Auto-mapping virtual flags are problematic**: Setting both `update_email` and `update_sms` to true in virtual mappings is a design liability. Future auto-mapping implementations should reconsider this approach.

3. **Logic requires explicitness for safety**: The current conditional checks were designed for single-channel-per-mapping scenarios. Any multi-channel or ambiguous mapping support requires logic redesign.

4. **Dead integrations should be cleaned up**: The Magento integration is not actively used but creates ongoing maintenance burden. Inactive integrations should be candidates for removal or deprecation.

5. **Hands-on implementation accelerates learning**: [Erik Andersson] emphasized that working directly with the code to fix real bugs is far more effective than walkthroughs for knowledge transfer.