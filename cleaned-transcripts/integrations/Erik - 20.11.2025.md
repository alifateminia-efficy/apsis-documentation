---
source_file: Erik - 20.11.2025.txt
domain: Integrations
topics: [Integration Uninstallation Flow, Clear Integration Endpoint, API Key Management, CRM System Errors, Webhook Cleanup, Force Deletion Patterns, Customer Account Deletion]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [FCC Enterprise, Generic Connector, Lime, Dynamics 365, Clear Integration Endpoint, Delete Integration Secret Key, Postman Collections, Integration Manager, Squid Proxy, RDS Database]
session_type: debugging-session
---

## Session Overview

Erik Andersson leads a knowledge transfer session with Michal Rosikiewicz and Tomasz Kowalski on the **Clear Integration** endpoint and the forced deletion workflow for customer integrations. The session covers why the forced uninstallation capability exists, how to use the specialized `Delete Integration Secret Key`, the security implications of this powerful endpoint, and walks through a live debugging scenario where a customer (Maltex Plastics) with a Lime integration needs to be forcefully deleted due to changed API credentials in their CRM system. The team successfully executes the deletion and discusses follow-up steps with the customer regarding webhook cleanup.

---

## Integration Test Collections and Debugging Resources

### FCC Enterprise Legacy Connector
[Erik Andersson]: FCC Enterprise is a **legacy connector** (not the generic connector). The team maintains a Postman collection containing all relevant FCC Enterprise API endpoints that they use for integration work.

The **FCC Enterprise collection** documents:
- How to retrieve all contacts
- Request structures for common operations
- Real example payloads for debugging

This is valuable when developers need to verify whether issues stem from receiving incorrect data versus handling received data incorrectly.

### Generic Connector Collection
[Erik Andersson]: The generic connector collection contains a complete endpoint for every capability available in the generic connector. You can configure which domain you're calling against and verify the result. This is particularly useful during development to isolate whether the problem is:
- Receiving malformed data from the external system, or
- Incorrect data handling in the connector

### Lime Legacy Connector
Lime is also a **legacy connector**. While not as comprehensive as FCC Enterprise, some example endpoints are documented in the collection.

### Dynamics 365 Collection
The Dynamics 365 collection documents all endpoints used to retrieve customer data. [Erik Andersson]: This is "very very useful" for debugging Dynamics customers because teams frequently need to:
- See how data appears in Dynamics for specific customers
- Check language variations (Swedish vs. English, etc.)
- Verify permissions
- Identify empty result sets

---

## The Clear Integration Endpoint: Purpose and Design

### Background: Installation and Uninstallation Refactoring
[Erik Andersson]: Two weeks prior to this session, the team had groomed an epic to refactor the installation and uninstallation flow. The original problem was that **functions were doing too much**. The team had to pass internal flags through 4-5 different function layers as parameters everywhere, creating maintenance burden.

The **Clear Integration endpoint** solves this by:
1. Calling the uninstallation function internally
2. Using the `ignore_errors` flag built-in
3. Eliminating the need to thread flags through multiple layers

### When Force Deletion is Necessary
[Erik Andersson]: Force deletion is used when a customer wants to uninstall an integration but the external system (CRM) is unreachable or misconfigured. Common scenarios:

- **API credentials have changed** — customer rotated keys in their Lime/Dynamics installation but didn't update them in Apsis
- **CRM configuration has changed** — the external system no longer grants necessary permissions
- **Network/firewall issues** — the CRM becomes unreachable

In these cases, the customer wants the integration removed from Apsis even if the uninstallation handshake with the CRM fails.

### The ignore_external_system_errors Flag
[Erik Andersson]: When this flag is enabled:
- The uninstallation function proceeds regardless of CRM errors
- All **Apsis database entries** are deleted
- All **stored credentials** are removed
- The customer can reinstall with new credentials if desired

> "We are failing to do the uninstallation request, but they want us to delete apps still and this would be a prime example where you would use this like forced on installation because here we are setting this ignore external system errors. So regardless if the CRM system is throwing an error in our face, we will still proceed in the flow and remove everything we have in in our database."

---

## Delete Integration Secret Key: Security and Access Control

### What the Key Does
[Erik Andersson]: The Clear Integration endpoint requires a special internal key called **Delete Integration Secret Key** found in AWS Secrets Manager. This key is the only authorization required to trigger the forced deletion flow.

[Michal Rosikiewicz]: *"This this key will work with all kinds of uh integrations."*

[Erik Andersson]: Yes — the key works universally across Dynamics, Lime, EDL, and any other integration type, because:
1. The key only authorizes the trigger call
2. The internal uninstallation function then loads the **customer-provided credentials** for the specific integration
3. Those credentials determine which system is actually called

> "So it doesn't matter which integration you are trying to do this for. Like it can be Dynamics or Lime or EDL because in the end we will load the relevant credentials for the system."

### Critical Security Warning
[Erik Andersson]: This key must be treated as **highly sensitive**:

> "And this is like something you would typically really not want to leak out, because in theory, if a customer had this, you can delete every integration, which is right now a bit suboptimal. So please don't leak this."

The key is stored in AWS Secrets Manager under: `Delete Integration Secret Key`

It is added to the request as an authorization header.

### Why Not Use Delegation Keys?
[Erik Andersson]: Currently, the Clear Integration endpoint uses this internal API key rather than customer delegation keys. [Erik Andersson notes]: *"Technically, if we were to do this really like, maybe this should be a delegation key, but this is something we slash you can change in the future to optimize this but."* This is a noted area for future optimization.

---

## Customer-Side Deletion Flow: API Key Rotation

### Updating Credentials Without Uninstallation
[Michal Rosikiewicz]: If a customer has simply rotated their API key in their CRM (Lime, Dynamics, etc.), they don't need to uninstall. They can update the credentials in Apsis and continue using the integration.

For example, in FSM to press 12.0, there is an **Update API Key** endpoint that allows the customer to provide new credentials.

### Forced Deletion vs. Credential Update
[Michal Rosikiewicz]: The workflow decision point:
- **If they want to keep the integration:** Update API key, then retry the normal uninstall or just continue operating
- **If they want to delete the integration:** Update the API key, then call the normal uninstall endpoint

---

## Legal and Data Privacy Considerations: The Webhook Risk

### The Core Problem: Orphaned Webhooks
[Erik Andersson]: If the Clear Integration endpoint forcefully deletes an integration from Apsis **without successfully clearing webhooks from the customer's CRM**, the CRM will continue sending webhook requests to Apsis endpoints with customer/contact data.

> "And this is very bad from a legal perspective. I mean, sure, for apps, this kind of doesn't matter because it's not our fault that someone is sending us random person sensitive data. It's not us that would be get in trouble, but we we don't want that to happen."

[Erik Andersson]: After force deletion, the webhooks will be rejected because the installation no longer exists in Apsis, but the data is still being transmitted over the network.

### Why Full Customer Self-Service Force Deletion Isn't Enabled
[Michal Rosikiewicz]: Proposed feature — expose a "Force Delete" button in the UI so customers can self-serve.

[Erik Andersson's reasoning against exposing this]:
1. If deletion fails on the CRM side (webhooks not removed), customer data continues flowing
2. Legal liability is unclear when customer-initiated force deletion leaves orphaned webhooks
3. The current manual process ensures human verification

[Michal Rosikiewicz]: *"I don't see any value in doing it ourselves manually. If something is failing and they know that they definitely want to remove it, why not to leave that possibility for them to force the deletion?"*

[Erik Andersson]: *"Absolutely."* The team agrees this feature could be added with a proper disclaimer.

### Proposed Solution: Force Delete with Disclaimer
[Erik Andersson]: The implementation would be straightforward:
1. Add a Force Delete button to the UI
2. Append query parameter `ignore_external_system_errors=true` to the uninstall endpoint call
3. Include disclaimer: **"Please manually remove all webhook configurations from your CRM system"**

There is no technical barrier to this; it's purely a risk/compliance decision.

### Detecting Orphaned Webhooks
[Michal Rosikiewicz]: After forced deletion, you can check in **Integration Manager** (logs) or **AltaSync Manager** to see if the customer's CRM is still sending data to Apsis.

[Erik Andersson]: The system does not display the actual data being sent (to avoid exposing sensitive PII), but requests can be observed in logs.

---

## Practical Execution: The Maltex Plastics Lime Integration Case

### Setup: Required Parameters for Clear Integration Endpoint

The Clear Integration endpoint requires three core parameters — *"the holy Trinity of being able to pinpoint one unique installation"*:

1. **Account ID** — the customer's account identifier
2. **Section ID** — the section/workspace within that account
3. **Integration ID** — the specific integration type (e.g., "Lime", "Dynamics365", "FSC_Enterprise_2")

[Erik Andersson]: These must be exact. For Lime, the integration ID is literally `"Lime"`. For other systems like FSC Enterprise 12.0, you must specify the full integration ID: `"FSC_Enterprise_2"`.

### Postman Collection Configuration
[Erik Andersson]: The collection itself specifies the base URLs (production, staging, APAC), but you must configure:
- Account ID
- Section ID  
- Integration ID
- Authorization header with the Delete Integration Secret Key

The URLs do not change, but the request path parameters must be set for each request.

### Historical Account ID Format
[Erik Andersson]: The Maltex Plastics customer predated the migration to UID-based account IDs. Their account ID is literally their customer name: `"Maltex Plastics"` (not a UUID).

### Live Debugging: Initial Failures

**Attempt 1: Wrong Account Name**
- Provided account: `"Meltex Plastics"`
- Error: `401 Unauthorized — provided credentials not valid`
- Root cause: Typo in account name

[Michal Rosikiewicz]: *"It's interesting that you got the 401 error though"* — the 401 was actually coming from an attempt to reach the Lime system, which propagated up.

**Attempt 2: Correct Account Name, Different Error**
- Corrected account to: `"Maltex Plastics"`
- Error: `404 integration was not found`
- Root cause: Still had wrong section or account ID combination

[Erik Andersson]: The endpoint silently fails with 404 rather than logging the lookup failure, making debugging harder.

**Attempt 3: Verified Against Database**
- Checked Integration Manager logs and account details
- Confirmed correct section ID: `16134`
- Confirmed correct integration ID: `Lime`
- Request proceeded successfully

### Successful Deletion with Squid Proxy Edge Case

[Erik Andersson]: During execution, the team encountered a known edge case:

> "Occasionally, very rarely, when you try to uninstall, the database refuses to delete that squid proxy entry like it just hangs on that request for whatever reason."

In this case:
- The clear integration endpoint returned an error (HTML gateway error from Lime due to misconfiguration)
- The **ignore_external_system_errors** flag was active, so deletion proceeded anyway
- The database entry for the squid proxy was successfully removed despite the external error
- All Apsis database entries were deleted
- All credentials were purged

**Result**: The integration was completely cleared from Apsis, but webhooks remain uncleaned in the customer's Lime installation.

### Post-Deletion Follow-up
[Michal Rosikiewicz]: After successful forced deletion, the support team (SOC) must contact the customer with instructions:

> "Please remove any lingering webhook configuration from your Lime CRM system."

This ensures they don't continue sending contact/person data to Apsis for an integration that no longer exists.

---

## Reasons for Using ignore_external_system_errors

[Erik Andersson]: The flag is necessary in specific scenarios:

1. **No users in account** — Cannot delete event listeners from Apsis Audience product if no admin users remain
2. **Account terminated** — Cannot exchange delegation keys with a closed account
3. **CRM misconfiguration** — External system is broken, unreachable, or has configuration errors (as with Maltex Plastics)

> "Or the account has been terminated so we can't exchange delegation keys. So that's why we have the ignore audience. Or if the CRM has some misconfiguration, that's why we have the ignore external system errors."

---

## Team Workflow: Customer Support Tracking

### Zero-Point Story Approach
[Michal Rosikiewicz]: The team uses a board tracking system where customer escalations/investigations are added as zero-point stories. This ensures:

1. **Visibility** — Multiple team members know issues exist
2. **Tracking** — Historical record of customer issues and resolutions  
3. **Knowledge building** — Can search past issues when similar problems arise
4. **Prioritization** — Can be pulled into the next sprint if needed

[Erik Andersson]: In smaller teams (3-4 people), a rotating help channel check works. But with the current team size, the zero-point story approach provides better organization.

### Story Closure and Communication
After resolution, the story is marked complete and customer communication is written to the help channel. [Michal Rosikiewicz]: *"I will write on this help channel and if I wrote written something stupid you can you can add something."*

---

## Postman Collection Environment Variables

The team noted gaps in the Postman collection's pre-configured variables. **Available environments documented**:
- `URL_prod` (for production clear integration)
- `APAC` (for APAC region)
- `stage_one` (for staging)

However, not all environment variables are pre-populated. Users must manually add:
- Account ID
- Section ID
- Integration ID
- Authorization header values

The collection itself stores the API key path but not the actual secret value (for security reasons).

---

## Pending Issues and Escalations

### 1. Unknown CRM Domains Request
A Dynamics customer has requested information about what domains the Apsis integration can be reached from, possibly for CORS policy configuration. The team was unclear on:
- Whether this is asking for endpoint domains
- Whether it's related to CRM plugin configuration
- What the customer actually needs to configure

[Erik Andersson]: *"I have no idea what's that I... I don't know if this has some end point inside the CRM or the plug in or what's."*

**Status**: To be discussed with customer on next business day for clarification.

### 2. CRM Implementation Partner Bypass Attempts
[Erik Andersson]: A CRM implementation partner (Denis Moi or similar) has reached out with unspecified integration issues but is attempting to bypass normal support channels.

**Action**: Erik is redirecting them to proper support flow, but the team expects to need investigation during the week.

---

## Key Takeaways

1. **Clear Integration is powerful but risky** — It can delete any integration globally using a single secret key. This key must be guarded as carefully as production database credentials.

2. **Forced deletion requires follow-up** — When using `ignore_external_system_errors`, always inform the customer to clean up webhooks on their CRM side. Orphaned webhooks create legal/compliance risk.

3. **The "holy trinity" of parameters** — Account ID, Section ID, and Integration ID must be exact. Typos will fail silently with 404 errors rather than helpful messages.

4. **Squid proxy edge case exists** — Rarely, the database hangs on deleting squid proxy entries during uninstall, but `ignore_external_system_errors` overcomes this.

5. **API key rotation vs. deletion** — Customers can update credentials without uninstalling. Only force delete if they truly want to remove the integration.

6. **Zero-point story tracking works** — Even when investigations are quick, logging them as zero-point stories provides visibility and historical knowledge for the team.

7. **Postman collection is reference, not complete** — The collections document available endpoints but require manual parameter configuration for each request.

8. **Webhook orphaning is the main risk** — After forced deletion, there's no automatic cleanup of the customer's CRM webhook configurations. This must be communicated separately.

---

## Unresolved Questions and Follow-ups

1. **CORS domains for Dynamics customer** — Unclear what the customer is actually asking for. Requires clarification before response.
   - Is this about endpoint domains?
   - Is this about the plugin calling back to Apsis?
   - Related to CORS policy configuration?

2. **Delegation keys optimization** — Team noted that Clear Integration should ideally use customer delegation keys instead of an internal API key, but this is a future enhancement.

3. **Self-service force delete UI** — Team agreed this is technically feasible but haven't finalized the decision on whether to expose it to customers, pending compliance review.