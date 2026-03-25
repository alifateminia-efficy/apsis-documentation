---
source_file: Erik from Dec 11th 2025.txt
domain: Apsis One Integrations
topics: [Consent 2.0 Migration, Event Listeners, Subscription Mappings, Error Handling, Tribe Integration, File Imports]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Event Listeners, Consent 2.0, Subscription Mappings, CRM Systems, Tribe Connector, Magento Connector, Apsis One API, Delegation Tokens]
session_type: debugging-session
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Inbound Flow, Outbound Flow, Tribe]
---

## Session Overview

This debugging session between Erik Andersson and Michal Rosikiewicz focuses on resolving integration errors related to the migration from Consent 1.0 to Consent 2.0 in Apsis One. The team identified that several customer accounts (specifically Magento and Tribe connectors) still have legacy global email unsubscribe event listeners that should have been removed during the migration. The session covers the architectural shift in how consent is handled, the specific errors being encountered, and potential approaches for remediation—including whether to proactively remove listeners via internal API tokens or ask customers to update their subscription mappings.

---

## Consent 1.0 vs. Consent 2.0: Architectural Shift

### Historical Consent 1.0 Approach

In Consent 1.0, the system used a simplified, global listener-based approach:

> "The way we handled the opt out messages were that we registered a global e-mail unsubscribe listener if you installed any connector. That handled consents inside the CRM system. So if we received the e-mail unsubscribe event then we sent on the unsubscribe or the opt out events opt out request to the CRM system."

**Key limitations of Consent 1.0:**
- Could only listen to email unsubscribe events
- Could not handle opt-in events because no global opt-in events were provided historically
- Used a single global email unsubscribe listener per connector installation, regardless of the number of subscriptions configured
- No ability to distinguish between different subscription types during event processing

### Consent 2.0 Mapping-Based Approach

Consent 2.0 fundamentally changed this architecture:

> "When Consent 2.0 was introduced, we moved away from this. So instead of registering one global e-mail unsubscribe event, we registered a listener for the assignment attribute for each of the subscription that you have a mapping to."

**How Consent 2.0 works:**
- Listens to **topic assignment attributes** (also called subscription assignment attributes) rather than generic unsubscribe events
- Checks a hidden attribute on the profile that belongs to the specific topic
- Creates individual event listeners for each subscription mapping configured (e.g., if you map subscriptions A and B, you get two listeners—one per subscription)
- Receives assignment attribute change events with a binary value (0 or 1 or 2) indicating consent status
- Routes the corresponding opt-in or opt-out request to the CRM system based on the attribute value

[Erik Andersson]: "When we receive that event, now we check like is it 0, 1 or 2 and then we send the corresponding opt in or opt out message to the CRM system."

---

## The Migration Problem: Orphaned Event Listeners

### Root Cause

The migration to Consent 2.0 was not fully completed for all accounts. Some customer installations (particularly Magento and Tribe) still retain the old global email unsubscribe event listeners that should have been removed.

[Michal Rosikiewicz]: "We know that everything now is migrated to Constant 2.0. We shouldn't even have this constant 1.0 events."

[Erik Andersson]: "This is something trailing like we should not care about these unsubscribe events at all because we don't handle the consent that way anymore."

### Why This Causes Errors

When the integration receives a Consent 1.0 unsubscribe event:
1. The event listener still exists on the account (legacy artifact)
2. The integration attempts to process the event through the new mapping-based logic
3. **The event has no corresponding mapping** because the new system works per-subscription, not globally
4. The system fails to find a matching subscription mapping and logs an error: "Source entity not found"

[Erik Andersson]: "The reason why this fails now is because like we are looping through, we are normally we're looping through the mappings and trying to find the correct mappings, but now we are failing to do that because like this event is not. We we don't handle it this way anymore."

### Scope of the Problem

This is **not a widespread issue affecting all customers**, but rather impacts accounts that:
- Had the global listener registered during Consent 1.0
- Never had subscription mappings configured (or had them removed)
- Are actively sending events that trigger the listener (file imports, form submissions, etc.)

[Erik Andersson]: "This is something on their installation or on their account like this is something. This is something trailing like we should not care about these. Unsubscribe events at all because we don't handle the consent that way anymore."

---

## Affected Customers and Error Patterns

### Known Affected Accounts

**Magento Customer** - Confirmed issue:
- Receives file import events that trigger the orphaned listener
- Has no subscription mappings configured
- Error message: "Keep constant message source entity not found"

**Tribe Customer (Hola Press)** - Secondary issue being investigated:
- Different error pattern: "K contact field host" attribute mapping issue
- May be related to incomplete attribute configuration rather than listener orphaning
- Errors appear singular/isolated rather than recurring, suggesting event-specific rather than systemic

**Professional Services/Internal Accounts:**
- Several "CPS" (Corporate Professional Services) internal accounts flagged in logs
- These are expected and acceptable as internal testing accounts

### Error Investigation Approach

When investigating these errors, check:
1. The event source (file import, form tool, direct API call, etc.)
2. The installation's current subscription mappings
3. Whether the listener should exist on the account
4. Whether the error is systemic (affecting all events) or isolated (specific events only)

---

## Remediation Strategies

### Option 1: Proactive Listener Deletion via Internal API

**Approach:** Remove the orphaned listener directly without customer interaction

**Implementation details:**
- Access the internal API token and delegation token stored in the installation table
- Use the Apsis One API to unregister the specific event listener
- No customer approval or involvement required

[Erik Andersson]: "We have that for every integration, so in theory we can do that. The question is if we should do it."

**Stored credentials location:**
```
Installation table:
- One API data (client credentials)
- Delegation data (token for generating access tokens)
```

**Advantages:**
- Resolves the problem immediately without customer action
- Aligns with the system's current Consent 2.0 architecture
- Can be automated for all affected accounts

**Considerations:**
- Delegation tokens are generated for profile update operations in asynchronous inbound flows; using them for listener management is a secondary use case
- Need to verify this won't interfere with normal integration operations

### Option 2: Customer-Initiated Mapping Creation

**Approach:** Ask affected customers to create subscription mappings in their integration configuration

[Michal Rosikiewicz]: "Should we ask them to create this mapping to fix it?"

**Why this alone is insufficient:**
- Creating new mappings will register new listeners for future subscriptions
- It will **not remove the existing orphaned listener**
- If the customer has no intent to use consent features, this adds unwanted configuration burden

[Erik Andersson]: "But it won't remove the existing unsubscribe listener."

### Option 3: Code-Level Error Suppression

**Approach:** Modify the integration code to treat missing mappings as warnings rather than errors for specific scenarios

**Implementation:**
```
If source == "file_import" 
  AND integration_id == magento 
  AND mapping_not_found:
    Log as warning instead of error
```

**Trade-off:**
- Suppresses symptoms without addressing root cause
- Prevents error notification but doesn't fix the underlying listener orphaning
- Should not be the primary solution

[Erik Andersson]: "This is more like a notification or a warning that we are not going to do anything about it because we don't find any mappings for it."

---

## Customer Communication Considerations

### What NOT to Tell Affected Customers

If a customer has no subscription mappings configured, they were not actually receiving consent updates even in Consent 1.0:

[Erik Andersson]: "Right now they do not have any consent mappings. So even if we would have had this unsubscribe flow still really working then. I mean, we would still not have sent any message to them because we don't know to which subscription in the CRM we would have registered it."

### Potential Communication Approach

If proactive contact is decided:
> "We've noticed your integration doesn't have consent mappings configured. With the recent Consent 2.0 migration, if you want to receive subscription/consent updates from Apsis One, you'll need to configure these mappings. Would you like us to help set this up?"

**Caveat:** This should only be sent if the customer is actively using consent features or asking about consent-related functionality.

---

## Technical Data Points

### Consent Attribute Requirements

When registering event listeners in Apsis One, the system requests the following data be included in events:
- Email address
- SMS number
- CRM ID
- Lead ID (where applicable)

The integration validates that these fields are present in incoming attribute data before processing events.

### Common Mistakes in Field Mapping

A recurring issue with file imports: customers mapping the CRM ID field to fields that should not be CRM IDs. This causes:
- Event processing failures
- Inability to locate the correct profile in the CRM system
- "Source entity not found" errors

[Erik Andersson]: "When they start mapping things to say CRM ID when they shouldn't. All of this is like a file import, but because the event listener is still there, of course we will receive those events."

### API Token Management

**Installation-level tokens:**
- Stored in the `installation` table
- Include Apsis One API data (client credentials)
- Include delegation data for token generation

**Available API endpoints:**
- Current state: Only GET event_listeners call is available in Integrations Postman collection
- **Gap identified:** No DELETE/unregister event listener endpoint currently documented in Postman collection
- Potential solution: Create internal API token directly for listener management if needed

---

## Unresolved Questions and Next Steps

### Questions Still Being Investigated

1. **Tribe K Contact Attribute Error (Hola Press account):**
   - What is the specific attribute that's missing or misconfigured?
   - Is this the same listener orphaning issue or a different field mapping problem?
   - Why is it appearing as a singular error rather than recurring?
   - Action needed: Further log investigation once AVS system is more stable

2. **Merger Account Secondary Error:**
   - Appears to be the same listener orphaning issue as Magento
   - Source is form_tool (form submissions) creating profiles with incomplete data
   - Need to confirm if this is the same root cause or separate issue

### Immediate Action Items

1. **Remove orphaned listeners for identified accounts:**
   - Magento customer (file import account)
   - Merger account (form tool account)
   - Determine if using internal API token approach is acceptable from architecture perspective

2. **Audit Magento installations:**
   - Query logs for past 30 days to identify all Magento accounts affected by this pattern
   - Determine which ones need proactive remediation

3. **API endpoint development:**
   - Consider adding DELETE event listener endpoint to Integrations Postman collection
   - If internal token usage is approved, document the process

4. **Tribe attribute investigation:**
   - Wait for AVS performance to stabilize
   - Search logs for all K contact field errors on Hola Press account
   - Determine if they are all the same attribute or different attributes
   - Identify the mapping configuration for this account

---

## Key Takeaways

1. **Consent 2.0 migration incompleteness:** The shift from global event listeners to per-subscription-mapping listeners was not fully completed for all customer accounts. Legacy listeners remain and cause errors when triggered.

2. **Root cause of errors:** Orphaned Consent 1.0 global listeners cause "source entity not found" errors because the new Consent 2.0 architecture expects subscription-specific mappings that don't exist for these accounts.

3. **Listener creation is mapping-driven:** In Consent 2.0, each subscription mapping configured by the customer automatically triggers creation of an event listener for that subscription's assignment attribute. This is fundamentally different from the pre-created global listener approach.

4. **Remediation is multi-faceted:** The problem requires both code-level handling (error classification/suppression) and customer-level action (removing old listeners via API, potentially asking for mapping configuration).

5. **Not all customers need consent mappings:** If a customer has no subscription mappings, they were never receiving consent updates, even in Consent 1.0. Unwanted error messages don't indicate missing functionality for them.

6. **Internal API tokens available:** The system already stores the necessary credentials (Apsis One API data and delegation tokens) at the installation level to allow proactive listener management without customer involvement, if the architecture team approves this approach.