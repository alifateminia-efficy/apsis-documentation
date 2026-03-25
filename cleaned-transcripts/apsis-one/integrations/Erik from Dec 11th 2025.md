---
source_file: Erik from Dec 11th 2025.txt
domain: Apsis One Integrations
topics: [Consent 2.0 Migration, Event Listeners, Integration Error Handling, Magento Integration, Tribe Integration, Attribute Mapping]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Consent 2.0, Event Listeners, Subscription Mapping, Apsis Audience, CRM Integration, Delegation Tokens, Installation Table, Postman Collection]
session_type: debugging-session
subdomains: [Architecture, Tribe Integration]
---

## Session Overview

Erik and Michal investigated critical issues stemming from the migration to Consent 2.0 in the integrations platform. The primary problem is that legacy global email unsubscribe event listeners from Consent 1.0 are still active on customer accounts but are no longer being handled by the integration logic, which now processes consent changes at the subscription assignment attribute level. The team explored solutions including listener removal via API access, code-level error suppression, and potential customer notification strategies. A secondary Tribe integration issue with K contact field mapping was identified but required further investigation before the session concluded.

---

## Consent 2.0 Migration: Event Listener Architecture Changes

### Historical Consent 1.0 Approach

In the Consent 1.0 system, the integration registered a **single global email unsubscribe event listener** when any connector was installed. This listener handled all opt-out events by:

1. Listening for email unsubscribe events at the account level (not per-subscription)
2. Receiving unsubscribe events and forwarding opt-out requests to the CRM system
3. Being unable to handle opt-in events because no global opt-in events were available

[Erik Andersson]: > "Historically, so we the way we handled the opt out messages were that we registered a global e-mail unsubscribe listener if you installed any connector. That handled consents inside the serum system. So if we received the the e-mail unsubscribe event then we sent on the unsubscribe. Or the opt out events opt out request to the CRM system"

### Consent 2.0 Listener Model

When Consent 2.0 was introduced, the architecture shifted fundamentally:

1. **Attribute-based listening**: Instead of one global listener, the integration now registers listeners for **subscription assignment attributes** (the hidden attributes that track consent status for specific topics)
2. **Per-subscription mapping**: For each subscription mapping configured in the consent subscription mapping tab, a dedicated event listener is created for that subscription's assignment attribute
3. **Binary consent checking**: When an assignment attribute event arrives, the integration checks if the attribute value is `0` or `1` (or `2`), then sends the corresponding opt-in or opt-out message to the CRM
4. **Selective processing**: Only subscriptions that have explicit mappings generate event listeners

[Erik Andersson]: > "When Consent 2.0 was introduced, we moved away from this. So instead of registering one like global e-mail unsubscribe event, we registered a listener for the assignment. Attribute for each of the subscription that you have a mapping to. So if you set up a mapping for subscription A&B in in the consent. On the consent subscription mapping tab then we registered listeners for changes on the attribute like the assignment attributes for subscription A&B"

### Root Cause of Current Failures

On accounts that have not completed full migration (particularly Magento customers), the old global email unsubscribe listeners still exist in the Apsis Audience system. When events arrive through these listeners, the integration attempts to process them using the new Consent 2.0 logic:

1. The integration loops through consent subscription mappings to find a matching mapping
2. No mapping is found because the event came through the old global listener, not through the new attribute-specific listeners
3. The integration fails with an error because it cannot determine which subscription to route the consent change to

[Erik Andersson]: > "The reason why this fails now is because like we are looping through, we are normally we're looping through the mappings and trying to find the correct mappings, but now we are failing to do that because like this event is not. We we we don't handle it this way anymore."

### Why This Wasn't Caught During General Migration

[Michal Rosikiewicz]: > "We know that everything now is migrated to Constant 2.0. We shouldn't even have this constant 1.0 events."

[Erik Andersson confirmed this was account-specific]: > "Exactly. So like this is something on their installation or on their account like this is something. This is something trailing like we should not care about these. Unsubscribe events at all because we don't handle the consent that way anymore."

This is not a global issue affecting all customers—it would have appeared in logs for every customer with an integration installed. It appears to be specific to certain accounts (particularly Magento) where the event listeners were not properly cleaned up during their migration.

---

## Technical Implementation Details

### Event Listener Registration Process

When setting up a consent subscription mapping in the UI (e.g., in Tribe's subscription mapping tab), the integration:

1. Saves the mapping configuration
2. Automatically registers an event listener in Apsis Audience for the corresponding subscription's assignment attribute
3. This listener is targeted to receive only changes on that specific attribute for that specific subscription

[Erik Andersson]: > "If I were to set up this mapping here, if I press save here now, then I will set up a event listener where I listen to changes on the attribute that belongs to the e-mail channel for. The default subscription, whereas previously as soon as you installed Tribe, I would register a global e-mail unsubscribe event."

### Apsis Audience Event Data Requirements

When registering event listeners in Apsis Audience, the integration requests that **all event messages include**:
- Email address
- SMS number
- **CRM ID**
- Lead ID (if applicable)

The integration validates that all required data is present in incoming events before processing:

[Erik Andersson]: > "whenever we register event listeners in to audience, we ask audience to. Uh, provide us with um. E-mail, SMS, CRMID and any potential like lead ID like to include that in the. Uh, in the events that is sent to us. And when we verify if we have all the data like we check in the incoming attribute data from audience and check like does it exist? Yes. No."

### Installation Data Storage

The integration stores per-installation credentials in an **installation table**:

```
Installation Table Fields:
- Apsis One API credentials (client key, etc.)
- Delegation token data (used to generate access tokens for asynchronous inbound/delta/full sync flows)
```

[Erik Andersson]: > "So here you see like for every for every installation we have the one API. AI data and then we have the the delegation data also that we can utilize to generate the the access. For it."

These delegation tokens are automatically generated when an integration is installed and are used for asynchronous communication with Apsis Audience to update profiles.

---

## Proposed Solutions for Legacy Listener Issues

### Option 1: Remove Event Listeners via API (with Customer Consent)

**Method**: Use the Postman collection's existing **GET event listeners** endpoint to retrieve all listeners, then create a DELETE endpoint to remove the legacy global listeners.

**Process**:
1. Ask customer for UI token via Apsis support (SOC)
2. Use token to authenticate API calls to remove the listener
3. Notify customer that the listener has been removed and request they verify their integration setup

**Pros**: Transparent to customer; they are informed of the action
**Cons**: Requires customer interaction; Postman collection currently lacks the DELETE method

[Michal Rosikiewicz]: > "I can see we only have their get event listeners. Call. So that should that that could be one possibility to add ourselves to after customer of course agrees to to. Uh, as a staff user on their account"

### Option 2: Remove Listeners Using Internal API Token (Automatic)

**Method**: Use the **internal delegation token** stored in the installation table to authenticate API calls without customer involvement.

[Erik Andersson]: > "we can also check here. Integration ID equals... we have that for every integration, so in theory we can do that."

Since the integration automatically generates a delegation token at installation time for asynchronous operations (delta sync, full sync), this same token could theoretically be used to remove legacy listeners.

**Pros**: No customer interaction required; immediate remediation
**Cons**: Requires consideration of whether automated listener removal is appropriate; potential security/audit implications

[Michal Rosikiewicz's position]: > "I think we should remove it even without asking customers."

[Erik Andersson]: Initially expressed caution about doing this without customer awareness, but the decision leaned toward automatic removal.

### Option 3: Suppress Error in Code (Partial Solution)

**Method**: Modify the integration code to treat missing consent mappings as a **warning or notification** rather than an **error** when processing Consent 1.0 events.

**Implementation**: When looping through mappings fails to find a match, log the event but do not raise an error condition.

**Pros**: Immediate relief from error logs
**Cons**: Does not remove the underlying listener; does not address the root cause; customers with zero consent mappings configured will never receive subscription consent updates

[Michal Rosikiewicz]: > "this is not really an error. This is more like a notification or a warning that we are not going to do anything about it because we don't. We don't find any mappings for it"

[Michal Rosikiewicz preferred the removal approach]: > "Yeah, this could be one of the solutions, but we still would like to get out, get rid of this listener, yeah."

---

## Customer Notification and Onboarding Strategy

### For Customers with No Consent Mappings

A key insight emerged: if a customer has **no consent subscription mappings configured**, they are not receiving any subscription consent updates anyway—whether the legacy listener existed or not.

[Erik Andersson]: > "I mean this is this is I'm not so sure what they would actually fix because right now they don't have any consent mappings. So even if we would have had this unsubscribe flow still really working then. I mean, we would still not have sent any message to them because we don't know to which subscription in the CRM we would have registered it."

[Michal Rosikiewicz's assessment]: > "So we should ask them to verify the integration and maybe add some mappings for subscription to create new event listeners. If they still want to receive subscription challenges. Because right now it is broken for them like after migration to Constant 2.0."

**Recommended approach**: After removing the legacy listener, SOC can proactively contact the customer:
- Acknowledge the Consent 2.0 migration
- Confirm they have no active consent mappings
- Ask if they intend to use subscription consent handling
- If yes: provide guidance on setting up consent subscription mappings
- If no: confirm they do not need this functionality

### Caveat: Lack of Mapping ≠ Broken Integration

[Erik Andersson clarified]: > "There has not really been any. Unsubscribe events... There has not really been any... because like we we can absolutely still ask SOC to contact them and say like hello, we note that you don't have anything. Are you sure this is the case? But it's it's it's it's not really... We should not. Yeah, yeah, OK, so like... We should not [force mappings on them]"

The absence of consent mappings is not inherently a problem if the customer simply does not need that feature. Removing the listener and confirming their intent is sufficient.

---

## Identification of Affected Customer Base

### Primary Issue: Magento Integration Customers

The legacy listener issue appears **specific to Magento integration installations**, not a global problem affecting all integrations.

[Erik Andersson]: > "like because Magento is, I mean like it's it's not a widely used connector we would take."

This narrows the scope: Only Magento customers need remediation for this specific issue.

### Secondary Issue: Mergular (Tribe Integration)

A second customer, **Mergular**, has a similar global event listener issue and requires the same removal process.

### Audit Plan: Log-Based Customer Discovery

The team planned to investigate logs from the past month to identify all customers affected by this pattern:

[Michal Rosikiewicz]: > "I also investigate how many other Magento customers we should fix by getting data from logs like from last one month because I think this will this will be similar for other. Uh, customers too."

### Confirmed Internal Accounts

The team identified several accounts in error logs that are **internal test/professional services accounts** and should be excluded from customer-facing remediation:
- CPS (Professional Services team internal account)
- Various merger/integration test accounts

---

## File Import as a Secondary Issue Source

### CRM ID Mapping Errors in File Imports

A secondary concern emerged regarding **file import operations**: when customers map file import fields to the **CRM ID** field when they should not, this causes problems in the integration.

[Erik Andersson]: > "When they start mapping things to say CRM ID when they shouldn't. All of this is like a file. Import, but because the event listener is still there, of course we will receive. We will receive those. Events."

**The issue**: File import events arrive as Consent 1.0 events through the legacy global listener. If the file import has mapped a non-CRM-ID field to the CRM ID attribute, the integration receives these false events and cannot process them (no mapping found).

**Mitigation**: This is handled partially by the error suppression approach, but ideally customers should not be mapping file import fields to CRM ID in the first place.

---

## Tribe Integration: K Contact Field Mapping Issue

### Problem Description

A singular error from Tribe integration customer **Hola Press** (installation ID 21641):

```
Message: "Keep constant message source entity not found"
Affected Field: K contact
Error Type: Appears to be attribute mapping related
```

### Investigation Status: INCOMPLETE

The team was unable to fully diagnose this issue by end of session. 

**Initial observations**:
1. This is a **singular error**, not recurring—which suggests it may have occurred during a specific file import operation rather than being a systemic configuration problem
2. The error appears related to a field mapping where the expected attribute is missing or not found
3. The team needed to:
   - Identify which attribute/mapping the K contact field refers to
   - Check if other K contact attributes have similar issues in logs
   - Determine if this is related to the Consent 1.0 listener issue or a separate problem

[Erik Andersson]: > "But this even more weird is that it is a singular error message because if this had been something for the whole installation then we would have seen more of them."

**Tools attempted**: 
- Apsis Attribute Management System (AVS) query to find the specific attribute mapping
- Log searching by installation ID, integration type, and attribute name

**Blocker**: AVS system was crashing during queries, preventing further investigation.

[Erik Andersson]: > "I hate AVS sometimes. I really, really hate AVS."

### Recommended Next Steps

1. Retry AVS queries after system stabilization
2. Check logs for other K contact attribute errors from the same installation
3. Determine if this is correlated with file import or other operations
4. If Consent 1.0 listener issue is also affecting Tribe, remediate that first and recheck logs

---

## Key Takeaways

1. **Consent 2.0 migration incomplete on some accounts**: Legacy global email unsubscribe listeners from Consent 1.0 remain active on Magento (and possibly other) customer accounts, causing integration failures because the new code expects per-subscription attribute listeners.

2. **Root cause is architectural mismatch**: The integration code now processes Consent 2.0 attribute changes but still receives events from Consent 1.0 listeners, creating a path where no mapping is found.

3. **Scope is limited**: Not a global issue; appears specific to Magento and possibly a few other integrations. All other customers are on Consent 2.0 properly.

4. **Removal of legacy listeners is the correct fix**: Whether done with customer consent (via UI token) or automatically (via internal delegation token), the old listeners should be deleted. Code-level error suppression is only a partial solution.

5. **Customers without mappings are not necessarily broken**: If a customer has zero consent mappings configured, they are not receiving consent updates regardless of the legacy listener. Notification should ask about intent, not force configuration.

6. **Internal delegation tokens are available**: The integration stores these in the installation table and uses them for async operations; they can be leveraged for automated remediation if approved.

7. **Tribe K contact field issue is unresolved**: A singular error suggests this may be import-specific, not systemic. Full investigation blocked by AVS system instability; requires follow-up.

8. **File import CRM ID mapping is a known pain point**: Customers should not map non-CRM-ID fields to the CRM ID attribute; this causes integration failures. May need stronger validation/guidance.

---

## Unresolved Questions & Action Items

### Immediate Actions Required

1. **Audit affected customer base**: Use logs from the past month to identify all customers with legacy Consent 1.0 global email unsubscribe listeners (particularly Magento, but check broadly)

2. **Build API removal capability**: Add DELETE event listener functionality to Postman collection or build internal removal method

3. **Decide removal approach**: Confirm whether to use internal delegation token (automatic) or request customer UI token (with notification)

4. **Implement code change**: Either remove listeners proactively OR add code to suppress these errors as warnings (decision pending approval)

5. **Investigate Hola Press/Tribe K contact issue**: Retry AVS queries, check logs for related errors, determine if it's import-specific or mapping-related

### Open Questions

- **Should listeners be removed automatically via delegation token without customer notification?** [Michal leaning toward yes; Erik expressed caution but deferred]
- **How many Magento customers are affected?** [Requires log analysis; not yet quantified]
- **Are there other integration types with the same issue besides Magento?** [Suspected but not confirmed]
- **What exactly is the K contact field mapping failure?** [Investigation incomplete due to AVS crash]
- **Should the integration validate that customers are not mapping file import fields to CRM ID?** [Identified as secondary issue; not yet prioritized]

### Follow-Up Session Needed

Michal indicated need to reconvene after lunch to continue investigation of the Tribe K contact field issue once AVS system stability is restored.