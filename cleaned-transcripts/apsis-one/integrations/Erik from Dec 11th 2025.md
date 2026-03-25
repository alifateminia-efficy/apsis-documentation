---
source_file: Erik from Dec 11th 2025.txt
domain: Apsis One Integrations
topics: [Consent 2.0 Migration, Event Listeners, Subscription Mappings, Magento Integration Issues, Tribe Integration, File Import Problems, CRM ID Mapping]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Event Listeners, Consent 2.0, Subscription Mappings, Global Email Unsubscribe Events, Assignment Attributes, Delegation Tokens, API Integration]
session_type: debugging-session
subdomains: [Inbound Flow, Duplicate profiles in Apsis]
---

## Session Overview

This debugging session focused on two critical issues in the Apsis One Integrations platform related to the Consent 2.0 migration. Erik and Michal identified and discussed problems with legacy event listeners still firing for accounts that migrated to Consent 2.0, specifically in Magento and Tribe connectors. The session covered the architectural differences between Consent 1.0 and 2.0 event handling, identified affected customer accounts, and explored remediation strategies including code-level fixes and manual listener removal via API calls.

---

## Consent 2.0 Migration: Architectural Changes

### From Consent 1.0 to Consent 2.0 Event Handling

**Erik Andersson:** In Consent 1.0, the system couldn't listen to attribute changes. With Consent 2.0, we now listen to topic assignment attributes or subscription assignment attributes. The key change is that you must check the consent status on the hidden attribute for the profile that belongs to the specific topic.

Before Consent 2.0, consent handling relied on a **global email unsubscribe listener** that was registered when any connector was installed. This listener would catch email unsubscribe events and send corresponding opt-out requests to the CRM system. However, the system could not handle opt-in events because there were no global opt-in events available historically.

### The New Listener Architecture

With Consent 2.0, the approach changed fundamentally:

- Instead of registering one global email unsubscribe event listener, the system now registers **a listener for the assignment attribute for each subscription** that has a mapping defined
- If mappings exist for subscription A and B in the consent subscription mapping tab, listeners are registered for changes on the assignment attributes for both subscriptions
- When an event is received, the system checks if the value is 0, 1, or 2, then sends the corresponding opt-in or opt-out message to the CRM system

**Erik Andersson:** > "Previously as soon as you installed Tribe, I would register a global email unsubscribe event. Listener for every subscription. Now we handle this immediately on the assignment attribute level instead."

---

## Magento Integration: Legacy Event Listener Problem

### Root Cause Analysis

**Michal Rosikiewicz:** We know that everything now is migrated to Consent 2.0. We shouldn't even have Consent 1.0 events.

**Erik Andersson:** Exactly. This is something on their installation or account—it's something trailing that we should not care about. These unsubscribe events shouldn't exist at all because we don't handle consent that way anymore. The reason this fails now is because we loop through mappings trying to find the correct matches, but we fail because this event is not something we handle anymore. This listener shouldn't exist on their account, which is why it's failing.

### How This Occurred

The issue stems from a missed migration step. When accounts migrated to Consent 2.0, the system did not remove the legacy global email unsubscribe event listeners that were created during Consent 1.0. Now when any event matching that listener fires, the integration code tries to find corresponding mappings (which may not exist), and the error is triggered.

**Michal Rosikiewicz:** > "This is like part of the migration to Consent 2.0 that somehow was missed in integrations because we still [are] listening to this global event and we cannot do anything about it."

### Customer Impact Assessment

Not all customers are affected. **Erik Andersson** noted: "This is not something global or you would have seen this for like every customer. Everyone that installs integration [would have it]. It's only certain accounts."

The affected accounts identified:
- At least two Magento customers with this issue
- Mergular account (also Magento-based, showing similar errors)

**Erik Andersson:** Magento is not a widely used connector, so the scope is limited.

---

## Proposed Solutions

### Option 1: Remove Event Listener via API

The cleanest solution involves removing the legacy event listener from affected accounts. This requires:

1. **Using delegation tokens** stored in the installation table to authenticate API calls
2. **Calling the Apsis One API** to unregister the listener

**Erik Andersson:** We have delegation tokens for every integration. In theory we can do this. The question is if we should do it without asking customers.

**Michal Rosikiewicz:** > "I think we should remove it even without asking customers."

The integration stores both the Apsis One API data and delegation token data in the installation table, which can be used to generate the necessary access credentials:

```
Installation table:
- One API key
- One API data
- Delegation token data
```

**Michal Rosikiewicz** noted that the current Integrations Postman collection only has a `GET event listeners` call. Adding a `DELETE event listeners` capability would enable automated cleanup.

### Option 2: Change Code to Skip Missing Mappings

An alternative approach is to modify the integration code to treat this scenario as a warning rather than an error:

**Erik Andersson:** > "We can swiftly resolve this error from our part by saying like this is not an error, this is a warning."

Since the event listener is still firing but no mappings exist, the integration finds nothing to act on and logs an error. Changing this to a warning would reduce noise while still allowing investigation.

**Michal Rosikiewicz:** "This could be one of the solutions, but we still would like to get rid of this listener."

### Option 3: Hybrid Approach

1. **Immediately remove** the legacy event listeners for all affected accounts using delegation tokens
2. **Check logs** from the past month to identify all Magento customers with this issue (not just the two currently visible)
3. **Notify affected customers** to verify their integration setup
4. Have **Support (SOC) contact them** to confirm whether they want consent functionality and help them set up proper subscription mappings if needed

---

## Customer Communication Considerations

**Michal Rosikiewicz:** Should we ask them to create mappings to fix it? Or should we do some migration and do it for them?

**Erik Andersson:** Setting up mappings won't solve the problem because they don't have any mappings right now. If they set up mappings themselves, we would register listeners for the corresponding subscriptions, but it won't remove the existing unsubscribe listener.

**Important note:** If customers didn't have consent mappings before the migration (or removed them), they likely had no active consent handling during Consent 1.0 either. Simply removing the listener won't break their integration—they just won't receive consent events unless they explicitly create subscription mappings.

**Erik Andersson:** > "There has not really been any [breakage]. If they didn't have any consent mappings before, like we have of course never removed any consent mappings during this transition."

The recommendation is to contact customers but not pressure them to act if they don't need consent functionality.

---

## Tribe Integration: K Contact Field Missing Mapping

### Issue Description

A singular error message appears for a Tribe installation where an inbound event references a K contact field attribute that has no corresponding mapping in the system.

**Error signature:**
- Source: Form Tool (creating profiles via forms)
- Missing mapping for K contact field attribute (ID: 07/6, mapping system field: 91491)
- Error: "Source entity not found"

**Erik Andersson:** When we register event listeners in Apsis One, we ask Apsis One to provide us with email, SMS, CRM ID, and any potential lead ID to include in the events sent to us. When we verify if we have all the data, we check in the incoming attribute data from Apsis One and verify: does the CRM ID exist? Do we have a lead ID?

### Key Observation

This appears to be a one-off error, not a recurring pattern:

**Erik Andersson:** > "This is even more weird is that it is a singular error message because if this had been something for the whole installation then we would have seen more of them."

The error occurred during a file import operation, suggesting it may be related to how attributes were mapped during the import process.

**Michal Rosikiewicz:** For this Tribe installation, the source is form tool creating profiles via forms. Other mappings show normal behavior, suggesting this is an isolated mapping issue rather than a systemic problem.

### Investigation Needed

**Michal Rosikiewicz** proposed checking logs for other K contact attributes on the same installation to determine if this is isolated or a pattern. The investigation was cut short due to platform issues (AVS crashing) and a scheduled break.

**Erik Andersson:** > "I don't fully know yet why this is. I need to check more."

---

## File Import as a Common Integration Problem

**Erik Andersson:** Issues like this with file import are a major culprit for problems in integration. When customers start mapping things to CRM ID when they shouldn't, it causes cascading failures.

The risk is especially high when:
- File imports are used without proper attribute mapping validation
- Customers map internal file fields to CRM ID fields
- Event listeners expect certain attributes (email, SMS, CRM ID, lead ID) but the import provides different data

---

## Internal Accounts vs. Customer Accounts

The investigation identified several error patterns across different account types:

- **Professional Services (PS) internal account:** Experiencing similar listener issues (expected to be OK as an internal test account)
- **Magento customer accounts:** Two distinct customers with legacy listener problems requiring remediation
- **Mergular account:** Similar Magento-based listener error pattern
- Other errors traced to legitimate form submissions and file imports

**Erik Andersson:** All of these [internal accounts] are personal accounts. The PS team account is internal and expected to have test configurations.

---

## Technical Implementation Details

### Event Listener Registration Flow

When an integration is installed, the system:
1. Stores API credentials and delegation tokens in the installation table
2. Registers event listeners with Apsis One specifying required attributes (email, SMS, CRM ID, lead ID)
3. For Consent 2.0, registers specific listeners per subscription mapping rather than global listeners

### Delegation Token Usage

**Erik Andersson:** The inbound flow is asynchronous. When you install an integration, we generate a delegation token that we utilize to communicate with Apsis One to update profiles for the delta sync flow and the full sync flow.

This delegation token can be reused to:
- Update profile data during syncs
- Remove orphaned event listeners
- Perform administrative operations on the account without requiring customer UI tokens

### Required API Capability Gap

Currently, the integration only supports querying event listeners. To implement automated cleanup, the system needs:

```
GET /integrations/{id}/event-listeners
DELETE /integrations/{id}/event-listeners/{listenerId}
```

---

## Key Takeaways

1. **Consent 2.0 Migration Incomplete:** The migration from Consent 1.0 to Consent 2.0 left legacy global email unsubscribe event listeners in place on some accounts (particularly Magento customers). These listeners fire but find no corresponding mappings, causing integration errors.

2. **Root Cause is Not Widespread:** This is not a global issue affecting all customers. The problem appears isolated to specific connector types (Magento) and accounts that lack consent subscription mappings.

3. **Two Viable Remediation Paths:**
   - **Recommended:** Use stored delegation tokens to automatically remove legacy listeners via API (requires adding DELETE capability to Postman collection)
   - **Alternative:** Downgrade to warning-level logging so errors don't alert unnecessarily

4. **Listener Removal Won't Break Customer Functionality:** Customers without active subscription mappings had no working consent flow anyway, so removing the listener is safe.

5. **File Import Mapping Validation Needed:** Separate from the Consent 2.0 issue, file imports that map to CRM ID incorrectly create cascading failures. This is an ongoing validation problem in the integration.

6. **Tribe K Contact Field Issue:** A singular mapping issue in Tribe appears isolated to one account's form-based profile creation. Further investigation needed to determine if this is a one-off configuration error or a pattern.

---

## Unresolved Questions & Action Items

### Immediate Actions Required

1. **Identify all affected accounts:** Query logs from the past month to find all Magento (and potentially other connector) customers with legacy email unsubscribe listener errors. Currently identified: 2-3 accounts.

2. **Implement listener removal capability:** Add DELETE event listener endpoint to Integrations Postman collection or create internal cleanup utility using delegation tokens.

3. **Execute cleanup:** Once tooling is in place, remove legacy listeners from affected accounts. Recommend doing this proactively without waiting for customer approval (given that the listener doesn't provide functional value).

4. **Customer notification (optional):** Have Support contact affected customers to verify consent requirements and offer to help set up subscription mappings if needed.

### Outstanding Questions

1. **Tribe K Contact Field:** Why does a single event reference an attribute (07/6) that exists but has no mapping (91491)? Is this a form import configuration issue or a mapping database corruption?

2. **Should we be more aggressive about removing orphaned listeners system-wide?** This appears to be an isolated issue, but should there be a periodic cleanup job?

3. **What other Consent 1.0 artifacts might exist?** If global unsubscribe listeners persist, do other Consent 1.0 structures remain?

### Session Continuation

The investigation into the Tribe K contact field issue and the Mergular account pattern was interrupted at 33:35 for a lunch break. **Erik Andersson** indicated he needed additional time to fully analyze the Tribe issue before drawing conclusions.