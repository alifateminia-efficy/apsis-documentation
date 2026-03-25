---
source_file: Erik from Dec 11th 2025 (continued).txt
domain: Apsis One Integrations
topics: [Consent Mapping, Magento Integration, Auto Mapping Logic, Consent Change Processing, Message Processing Pipeline]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Consent Mappings Client, Message Processor, Integration Manager, Auto Mapping Feature, Subscription Channels (Email/SMS)]
session_type: debugging-session
subdomains: [Architecture, Different Types of Connectors, Outbound Flow, Magento Integration]
---

## Session Overview

This debugging session focused on identifying and resolving a critical issue in Magento consent mapping handling. Erik Andersson walked through a bug where consent change messages for Magento integrations were being silently skipped due to a logical flaw in the message processor's consent mapping validation. The team discovered that Magento's use of external consent mapping (via the `auto_mapping` property) combined with a faulty conditional check was causing consent updates to be dropped entirely. The session concluded with agreement to fix the logic and potential follow-up action items regarding Magento customer outreach.

---

## Understanding Consent Mapping Architecture

### Standard Consent Mapping Approach

Most integrations handle consent mapping by configuring subscription mappings on the integration page. For example, in Tribe:
- Product mappings (e.g., "home") are mapped to email/SMS channels
- These mappings are stored and managed within the Apsis integration system

**[Erik Andersson]:** "Most integrations that we have, we handle by setting up consent mappings on the integration page... for example in tribe I would have like the products home and mapped to the e-mail e-mail channel for the default subscription."

### Magento's Different Approach

Magento fundamentally differs from other CRM systems: **it handles consent mapping internally within Magento itself**, not within Apsis One's integration page.

**[Erik Andersson]:** "This is not the way it works in Magento, because for Magento, handles the consent mapping inside Magento. You do not set this mapping up here."

---

## The Consent 2.0 System and Auto Mapping

### How Consent 2.0 Works with Mappings

After Consent 2.0, the system listens to changes on **assignment attributes for channels**:
- When a user opts in/out on the email channel for default subscription, an event is received inside the integration
- The system needs to know which subscriptions users are interested in to process these events correctly

### Auto Mapping Property for External Consent Systems

Because Magento handles consent mappings internally (not in Apsis), the system doesn't know which subscriptions users are interested in. To solve this, there is a property called `auto_mapping` on the installer option:

```
automatic_consent_mappings: true
```

**[Erik Andersson]:** "That is that because the mappings are not handled inside of integrations, we don't know what subscriptions they are actually interested in. That's why for such integrations that handle the consents inside of the CRM system, there is a property called auto mapping."

### Behavior During Installation

When `automatic_consent_mappings` is enabled during installation in the integration manager:
- Event listeners are registered for **all subscriptions** in the section
- **No actual consent mappings are set up on Apsis's side**
- Virtual consent mappings are generated at runtime to make the flow work

---

## The Bug: Double-Condition Consent Validation Logic

### Error Symptoms

Consent change messages for Magento were failing with this error:

> "We are skipping the consent message because we could not find any corresponding source entity for the consent."

This occurred in the **Message Processor at line 426**.

### Root Cause Analysis

The message processor validates consent mappings using a flawed conditional structure:

```
if (mapping.topic_id matches subscription AND channel is email):
    if (mapping.update_email == true AND mapping.update_sms != true):
        set source_entity
else if (mapping.topic_id matches subscription AND channel is sms):
    if (mapping.update_sms == true AND mapping.update_email != true):
        set source_entity
else:
    source_entity remains empty → ERROR
```

**The Critical Problem:** When `auto_mapping` is enabled, the virtual consent mappings are generated with **both `update_email` and `update_sms` set to true** (because the system doesn't know which channels the user is interested in).

This means:
- A mapping with `update_email=true` AND `update_sms=true` would **fail both conditions**
- Neither condition evaluates to true (each requires the other channel to be false)
- The source entity remains empty, and the consent message is silently dropped

**[Erik Andersson]:** "Now, unless you have already found it, I'm gonna give you like 30 seconds to try and find like what happens here if we have update SMS and update e-mail set on the consent mapping... [After a pause] This would be like this row here like one row, but it is both true and it is also false. And because of this then we are like just completely ignoring this consent update message."

### Why This Bug Wasn't Caught Earlier

This is a Magento-specific issue because:
1. Only Magento uses `auto_mapping` with virtual consent mappings that have both flags set to true
2. For other integrations, each mapping is created individually with explicit channel selection—never both at once
3. Apsis only has **two customers with Magento installations, and neither is actually using the integration**

**[Erik Andersson]:** "I checked with Opera and like while we have two customers with a Magento installation, they are not actually using it. It's essentially a dead integration for them."

---

## The Virtual Consent Mapping Generation

### How Auto Mappings Generate Virtual Mappings

In the `consent_mappings` function, there's a special clause for `auto_mapping` integrations:

```
if (integration.auto_maps):
    // Generate virtual consent mapping for every audience item
    for each audience:
        virtual_mapping = {
            source_entity: "contact",
            topic_id: <from audience>,
            update_email: true,
            update_sms: true  // Set both because we don't know user preference
        }
```

**[Erik Andersson]:** "We set up like virtual consent mapping just so our flow will actually actually work... We export like everything that is inside the audience and we return the map... we can see like we are setting the source entity to contact and we're setting like the topic ID and we're saying like yeah this should. This mapping is like you can update the e-mail with it and you can update the SMS with it because we don't know if they are interested in e-mail or SMS or both."

Both flags are set because Apsis doesn't know whether the user is interested in email, SMS, or both.

---

## The Fix

### Simple Solution

Remove the restrictive double-condition logic. Instead of requiring the opposite channel to be false:

```
// Current (broken):
if (topic_matches AND channel is email AND update_email AND NOT update_sms):
    set source_entity

// Fixed:
if (topic_matches AND channel is email AND update_email):
    set source_entity
```

The presence of other flags on the mapping doesn't matter—we only care that the specific channel being updated is enabled in the mapping.

**[Erik Andersson]:** "The fix for this is obviously very simple, like we should just remove this close like if if the update is for e-mail and the mapping is for update e-mail then we should set the source entity."

### Why This Fix Is Safe

For non-auto-mapping integrations (all others except Magento):
- Each mapping is created with explicit channel selection
- A single mapping never has both `update_email` and `update_sms` set to true
- The fix will not cause any regressions

**[Erik Andersson]:** "Because for every mapping you set up, you specify if it is e-mail or SMS, we will never run into a situation where it is true for both of them. Like one single mapping will not have both e-mail and SMS unless like we had manually entered it. So this has not been an issue for any other integration."

---

## Forward-Looking Considerations

### Future-Proofing the Fix

Even though Magento isn't actively used, fixing this now ensures the logic works correctly if:
1. A new Magento customer is onboarded in the future
2. New CRM systems with similar "unknown preference" requirements are added
3. Consent mappings are ever redesigned to support multiple channels per mapping

**[Erik Andersson]:** "More importantly, if you were to implement a new CRM system in the future that has like something similar... that's why we actually should fix this just to have it properly working, even though it is technically not a problem for any other real CRM systems."

### Magento Customer Outreach Decision

The team discussed whether to contact the two Magento customers currently using dead integrations to remove them. **[Erik Andersson]:** "Let me talk with Opera about it" — this will be handled as a separate discussion with the operations team.

---

## Work Planning and Ticketing

### Scope of Work

The fix requires approximately **4 separate, independent stories** to fully address all Magento consent-related issues:
- Core double-condition logic fix
- Auto mapping validation improvements
- Potential customer communication regarding dead integrations
- Additional Tribe consent issues (deferred to later)

**[Erik Andersson]:** "I think that is like 4-4 separate stories ish that would be needed to get this working and all of them are quite independent on each other."

### Implementation Plan

- **Erik Andersson** will prepare stories and create tickets before his handoff
- **Tomasz Kowalski** is primary implementer but will be out December 12-13
- Erik aims to have at least 2 stories ready for Tomasz to start with upon return
- Hands-on implementation will be valuable learning opportunity beyond walkthrough knowledge

**[Erik Andersson]:** "I am looking forward to actually get some hands on because I think that's when you will really learn about it and how it is all connected. It is one thing sitting and listening to my boring walkthroughs, but it is a complete other thing, like actually having to fix things in the system."

### Additional Pending Issues

Michal mentioned separate ongoing issues not addressed today:
- **SQS and backoff strategy** — 6 months of constant alarms without resolution; requires logic changes
- Scheduled for discussion on Tuesday (or after Tomasz returns if Tuesday has guest conflicts)

---

## Key Takeaways

1. **Magento's external consent mapping creates a unique edge case** where virtual consent mappings have both `update_email` and `update_sms` set to true, causing a logical contradiction in the current validation code.

2. **The double-condition check is unnecessarily restrictive** — it should validate each channel independently rather than requiring the opposite channel to be false.

3. **The bug wasn't caught because Magento is effectively a dead integration** for current customers, but it represents a real issue for future implementations with similar external-mapping requirements.

4. **The fix is simple and low-risk** — other integrations create explicit single-channel mappings, so they're unaffected by the logic change.

5. **This is a good example of why defensive coding matters** — even for systems not currently in use, proper logic ensures future maintainability and prevents issues when the system is reactivated.

---

## Unresolved Questions & Action Items

| Item | Owner | Status |
|------|-------|--------|
| Contact Magento customers about removing dead integrations | Erik Andersson (coordinating with Opera) | Pending |
| Create 4 ticket stories for Magento consent fix | Erik Andersson | In Progress (targeting 2 ready before Tomasz returns) |
| Investigate Tribe consent issues | Erik Andersson | Deferred |
| SQS and backoff strategy discussion | Michal Rosikiewicz (schedule for Tue or after Dec 13) | Pending |
| Deploy consent mapping validation fix | Tomasz Kowalski | Waiting for stories (after Dec 13) |