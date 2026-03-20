---
source_file: Erik from Dec 11th 2025.txt
domain: Apsis One - Integrations
topics: [Consent 2.0 Migration, Event Listener Management, File Import Issues, Magento Integration, Attribute Mapping, CRM Integration]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Consent 2.0 System, Event Listeners, Subscription Mappings, Magento Connector, Tribe Integration, File Import, Assignment Attributes, Delegation Tokens, AVS (Audience Visualization System)]
session_type: debugging-session
---

## Session Overview

This knowledge transfer session covers a critical migration issue discovered in the Integrations domain following the transition from Consent 1.0 to Consent 2.0. The discussion centers on orphaned event listeners from the legacy consent system that are still active in customer accounts, causing integration failures. The team identifies affected Magento customers and explores both immediate remediation strategies (removing listeners and downgrading errors to warnings) and longer-term solutions (customer communication and mapping verification). A secondary issue with Tribe integration attribute mapping is introduced but left unresolved pending further investigation.

---

## Consent 1.0 vs. Consent 2.0 Architecture: Historical Context and Migration

### Legacy Consent 1.0 Event Handling

[Erik Andersson]: In the previous integration system before Consent 2.0, the platform could not listen to attribute changes directly. The entire consent workflow relied on a single global mechanism:

- A **global email unsubscribe event listener** was registered when any connector (like Magento) was installed
- This listener captured all unsubscribe events from the system
- When an unsubscribe event was received, the integration would send an opt-out request to the CRM system
- The system could NOT handle opt-in events because no global opt-in events were emitted by the platform

This approach was fundamentally limited because it provided only one-directional (opt-out) consent management.

### Consent 2.0 Architecture

[Erik Andersson]: Consent 2.0 introduced a fundamentally different architecture based on per-subscription attribute listening:

- Instead of registering one global unsubscribe listener, the system now registers **individual event listeners for each subscription mapping**
- The listeners monitor **topic assignment attributes** or **subscription assignment attributes** specific to each subscription
- When an attribute change event is received, the system checks the assignment value (0, 1, or 2) to determine the consent state
- The system then sends the corresponding opt-in or opt-out request to the CRM system
- This approach enables bidirectional consent management (both opt-in and opt-out)

> "When Consent 2.0 was introduced, we moved away from this. So instead of registering one like global e-mail unsubscribe event, we registered a listener for the assignment attribute for each of the subscription that you have a mapping to."

### The Migration Gap

[Erik Andersson]: The critical issue discovered is that the migration from Consent 1.0 to Consent 2.0 was incomplete for some customer installations. Accounts still retain their legacy global email unsubscribe event listeners even though the system no longer processes these events.

[Michal Rosikiewicz]: This represents a missed step in the migration process. While the Consent 2.0 infrastructure was deployed globally, the cleanup of legacy listeners was not completed for all installations, particularly Magento customers.

---

## The Magento Customer Issue: Orphaned Event Listeners

### Problem Description

[Erik Andersson]: A Magento customer account is experiencing integration failures because:

1. The legacy global email unsubscribe event listener still exists on their account
2. This listener was created during the Consent 1.0 era
3. Events matching this listener are still being sent to the integration system
4. However, the integration code now only knows how to process events through the new Consent 2.0 subscription mapping architecture
5. When events arrive that don't match any existing mappings, the system fails

[Erik Andersson]: The failure occurs during the mapping lookup process:
> "The reason why this fails now is because like we are looping through, we are normally we're looping through the mappings and trying to find the correct mappings, but now we are failing to do that because like this event is not. We we we don't handle it this way anymore."

### Root Cause Analysis

[Michal Rosikiewicz]: The customer does not have any Consent 2.0 subscription mappings configured. This creates a scenario where:

- The old unsubscribe listener fires when file import events trigger email unsubscribe actions
- The integration receives the event but finds no matching subscription mappings
- The code treats this as an error and logs a failure
- No consent action is taken because the system cannot determine which CRM subscription to target

[Erik Andersson]: The issue is compounded by the fact that:
> "If they didn't have any consent mappings before, like we we didn't, we have of course never removed any consent mappings during this transition, so...There has not really been any [impact]"

If the customer never had mappings configured, they likely never received consent messages even during Consent 1.0 operation.

---

## Proposed Solutions: Technical Implementation

### Solution 1: Remove the Orphaned Event Listener

[Michal Rosikiewicz]: The cleanest solution is to delete the legacy global email unsubscribe event listener from affected customer accounts.

[Erik Andersson]: This requires either:

**Option A: Customer-Initiated Removal via Support**
- Support contacts the customer and provides instructions to remove the listener from their Apsis One account
- Requires the customer to log in and manually remove the listener
- Alternatively, SOC (Support Operations Center) can be granted access to the customer account and remove the listener directly
- A new REST API call would be needed to the Apsis One platform to unregister the listener

[Michal Rosikiewicz]: Currently, the integration's Postman collection only has a "get event listeners" endpoint. To automate removal, the team would need to add a delete endpoint.

**Option B: Automated Removal via Internal Delegation Token**
- The integration system already stores delegation tokens for every installation in the database
- These tokens are used for asynchronous inbound flow operations (delta sync, full sync)
- The system could use these stored credentials to automatically unregister the listener without customer involvement
- This avoids requiring customer action and SOC intervention

[Erik Andersson]: The system already has access to the necessary credentials:
> "We have that for every integration...in theory we can do that. The question is if we should do it"

The delegation token approach is feasible and already used for other automated operations like profile updates during synchronization.

### Solution 2: Suppress the Error in Code

[Erik Andersson]: As a complementary measure, the integration code can treat this scenario as a warning rather than an error:

> "Like we can swiftly resolve this error from our part by saying like this is not an error, this is a warning."

When the system receives an event from a listener but finds no matching mappings (no mappings exist for that listener), it should:
- Log a warning message instead of an error
- Continue processing without failing
- Not attempt to send a consent message to the CRM (because there's no valid target)

This prevents false-positive error alerts while the listener removal is being organized.

[Michal Rosikiewicz]: This is a temporary mitigation while the listener cleanup is executed.

### Solution 3: Customer Communication and Verification

[Michal Rosikiewicz]: Even after removing the listener, the team should:
- Ask the customer to verify their integration configuration
- Encourage them to set up subscription mappings if they want to handle consent updates
- Clarify that their integration is not fully configured for Consent 2.0

[Erik Andersson]: However, this must be communicated carefully:
> "We can absolutely still ask SOC to contact them and say like hello, we note that you don't have anything. Are you sure this is the case? But it's it's it's not really [critical]"

If the customer has no mappings, they likely have no consent requirements and should not be alarmed by the removal.

---

## Scope of the Issue: Identifying Affected Customers

### Affected Accounts Identified

[Michal Rosikiewicz]: Log analysis revealed at least two customer accounts with this issue:

1. **Magento customer** (primary issue discussed)
2. **Another Magento customer** (marked as "palm mark" in logs)

Additionally identified:
- **CPS account** (noted as "Professional Services team" - internal account, safe to ignore)
- Other internal accounts

[Erik Andersson]: This is not a widespread issue affecting all customers:
> "Because everyone that installs integration [would be affected], if this were global. [But] we would have seen this for like every customer"

The fact that only Magento customers are affected suggests the issue is specific to that connector or to a particular installation cohort.

### Recommended Log Analysis

[Michal Rosikiewicz]: To identify all affected customers:
- Query integration error logs from the past month
- Search for listener registration errors related to Magento installations
- Filter for "missing consent mapping" or similar error patterns
- This will reveal other Magento customers requiring the same fix

---

## Technical Details: Event Listener Registration and Storage

### Listener Registration Process

[Erik Andersson]: When a subscription mapping is configured in the Consent 2.0 system:

> "If I were to set up this mapping here, if I press save here now, then I will set up a event listener where I listen to changes on the attribute that belongs to the e-mail channel for the default subscription"

The system:
1. Identifies the target subscription (e.g., "default subscription")
2. Identifies the email channel assignment attribute
3. Registers an event listener for that specific attribute with Apsis One's event system

### Listener Storage and Credentials

[Erik Andersson]: Installation data is stored in the database with the following critical information:

- **One API key/data**: Credentials for API access to Apsis One
- **Delegation token data**: Separate credentials used for asynchronous operations like profile synchronization
- **Integration ID**: Unique identifier linking the integration record to the Apsis One account

The database schema includes both the one-time API credentials and delegation tokens that can be used to interact with the Apsis One event system.

### Event Payload Requirements

[Erik Andersson]: When registering event listeners with Apsis One, the integration specifies which data fields should be included in event payloads:

- Email address
- SMS phone number
- CRM ID
- Lead ID (where applicable)

The integration then validates incoming events to ensure these required fields are present:
> "When we verify if we have all the data like we check in the incoming attribute data from audience and check like does it exist? Yes. No. Like do we have a CRM ID in for this profile? Do we have a lead ID?"

If required fields are missing, event processing may fail.

---

## File Import and CRM ID Mapping Issues

### Known Problem: CRM ID Mapping in File Imports

[Erik Andersson]: File imports are a major source of integration problems. A critical mistake occurs when customers map file import fields to the **CRM ID** field:

> "When they start mapping things to say CRM ID when they shouldn't. All of this is like a file import, but because the event listener is still there, of course we will receive those events."

This is problematic because:
- The CRM ID should be a system identifier from the target CRM
- When file import operations map to CRM ID, events are triggered
- The orphaned listener catches these events
- The integration then has no valid mapping to process them

[Michal Rosikiewicz]: The current errors in logs appear to be triggered by file import operations combined with the orphaned listener issue.

---

## Tribe Integration: Attribute Mapping Mystery (Unresolved)

### Issue Description

[Michal Rosikiewicz]: A second issue was identified in the Tribe integration concerning the **K contact field host** attribute.

[Erik Andersson]: When investigating this error, the system:
1. Receives an event from Tribe
2. Attempts to match the event to a subscription mapping
3. Fails to find a mapping for the attribute in question
4. Logs a specific error with mapping system field reference number `07/6`

[Erik Andersson]: The investigation revealed unusual characteristics:
> "This even more weird is that it is a singular error message because if this had been something for the whole installation then we would have seen more of them."

Only one error has been logged for this attribute mapping, which suggests:
- This may be a one-time event that triggered an edge case
- The attribute mapping may have been corrected afterward
- Or there's a specific condition that only sometimes causes this error

### Investigation Challenges

[Erik Andersson]: The investigation was hampered by the AVS (Audience Visualization System) database tool:
> "I hate AVS sometimes. I really, really hate AVS."

AVS crashes or becomes unresponsive when querying the specific customer's mapping data, making it difficult to determine:
- Which attribute the K contact field host maps to in the customer's configuration
- Whether the mapping exists at all
- What time the error actually occurred

[Michal Rosikiewicz]: A more detailed investigation is needed after a break to:
- Query Tribe integration logs for the specific customer (Hola Press ID 21641)
- Search for other K contact attribute errors from the same account
- Determine if this is part of a pattern or an isolated incident

### Open Questions

- What is the K contact field host attribute used for?
- Why is the mapping lookup returning null for this attribute?
- Did the mapping configuration change after the error occurred?
- Are there other similar errors for different K contact attributes from this customer?

---

## Key Takeaways

1. **Consent 2.0 Migration Incomplete**: The migration from Consent 1.0 to Consent 2.0 did not clean up legacy global email unsubscribe event listeners in some customer accounts, particularly Magento installations. These orphaned listeners continue to receive events that the new system architecture cannot process.

2. **Identify and Remove Listeners**: At least two Magento customers should have their orphaned listeners removed immediately. This can be done either via automated API calls using stored delegation tokens or via SOC support access. Log analysis should be run to identify all affected customers across the install base.

3. **Dual Mitigation Approach**: Two complementary fixes should be implemented:
   - **Code change**: Downgrade errors to warnings when events arrive for non-existent mappings (this is expected behavior for accounts without Consent 2.0 mappings configured)
   - **Operations**: Remove the actual listener from affected accounts to prevent events from being generated

4. **Customer Communication Strategy**: When cleaning up listeners, contact affected customers to verify their integration setup. However, do not require immediate action if customers have no subscription mappings—this may indicate they have no consent requirements.

5. **Architectural Lesson**: The transition from Consent 1.0's global listener model to Consent 2.0's per-subscription listener model required both code changes AND data cleanup. Future major migrations should include explicit cleanup steps and verification checks.

6. **File Import Risk Factor**: Customers who map file import operations to the CRM ID field create events that trigger integration flows. This is a known problematic pattern that should be documented in customer guidelines.

7. **Tribe Integration Needs Further Investigation**: The K contact field host attribute mapping issue requires more investigation due to AVS tool limitations. This should be revisited with better logging or alternative database query tools.

---

## Unresolved Questions and Action Items

### Immediate Actions Required
- [ ] **Generate log report**: Query integration error logs from past 30 days to identify all customers affected by orphaned consent listeners
- [ ] **Implement code change**: Downgrade "listener not found in mappings" errors to warnings in the integration codebase
- [ ] **Automate listener removal**: Develop or enhance the API endpoint to allow automated unregistration of event listeners using stored delegation tokens
- [ ] **Execute cleanup**: Remove orphaned listeners from identified Magento customer accounts
- [ ] **Customer outreach**: Contact affected customers via SOC to inform them of the listener removal and encourage them to verify/configure their subscription mappings

### Investigation Needed
- [ ] **Tribe K contact issue**: Further investigation of the K contact field host attribute mapping error after obtaining better access to customer configuration data
- [ ] **AVS reliability**: Document AVS tool limitations and explore alternative methods for querying customer mappings (possibly direct database queries or logs)

### Architectural Questions
- What other legacy listeners or configurations might exist from older system versions?
- Should migration checklist include automated verification of listener cleanup?