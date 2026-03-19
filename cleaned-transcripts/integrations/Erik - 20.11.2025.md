---
source_file: Erik - 20.11.2025.txt
domain: Integrations
topics: [Integration Deletion and Cleanup, API Key Management, Postman Collections for Testing, Uninstallation Flow, Webhook Management, Customer Support Workflows, Error Handling in Integrations]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Clear Integration Endpoint, FCC Enterprise Connector, Generic Connector, Lime CRM, Dynamics 365, Delete Integration Secret Key, Secrets Manager, Integration Manager, RDS Database, Webhook Configuration]
session_type: debugging-session
---

## Session Overview

This session focused on a customer support case involving the forced deletion of a Lime CRM integration that failed during normal uninstallation due to API credential changes. Erik walked Michal and Tomasz through the **Clear Integration** endpoint and its critical `ignore_external_system_errors` flag, demonstrating the complete workflow of using the **Delete Integration Secret Key** to forcefully remove integration data from the Apsis side while acknowledging that manual cleanup must occur on the customer's CRM system. The team successfully executed the deletion for the Meltex Plastics account while documenting the operational procedures and security considerations involved.

---

## Integration Testing Collections and Documentation

### Available Postman Collections for Different Connectors

Erik maintains comprehensive Postman collections for testing various integration endpoints:

- **FCC Enterprise Collection**: Contains endpoints for all legacy connector functionality (e.g., getting all contacts). This is a legacy connector, not the general connector.
- **Generic Connector Collection**: Provides a full suite of endpoints where you can configure which domain you're calling and verify the result. This is useful for development to isolate whether issues stem from receiving incorrect data or from incorrect handling of correctly received data.
- **Lime Collection**: Contains example endpoints (not a complete suite, as Lime is also a legacy connector).
- **Dynamics 365 Collection**: Contains all endpoints used for retrieving data from Dynamics. This collection is very valuable for debugging customer issues, particularly when needing to verify data formatting across different language configurations (e.g., Swedish vs. English), permissions, and result sets.

These collections serve as both testing tools and debugging aids for the support team.

---

## The Clear Integration Endpoint and Forced Deletion Workflow

### Rationale for the Clear Integration Endpoint

Previously, the installation and uninstallation flow had a significant architecture problem: **functions were doing too much and required passing multiple flags through four or five different function layers as parameters**. This created a maintainability nightmare.

[Erik Andersson]: The **Clear Integration** endpoint was introduced as a solution to this problem. It calls the uninstallation function but uses special **`ignore_errors` flags** to handle scenarios where the external CRM system cannot be reached or is misconfigured.

### The Key Use Case: Forced Deletion

When a customer wants to uninstall an integration but the uninstallation fails (e.g., due to changed API credentials, configuration changes, or permission issues), the customer may want to proceed with deletion anyway. The Clear Integration endpoint enables this by:

1. Setting the `ignore_external_system_errors` flag
2. Removing all database entries from the Apsis side
3. Removing stored credentials
4. Allowing the customer to reinstall with new credentials if needed

[Erik Andersson]: The key difference between the normal uninstallation endpoint and the Clear Integration endpoint is that **Clear Integration uses an internal API key** rather than requiring access to the customer's account credentials.

---

## Authentication and the Delete Integration Secret Key

### The Delete Integration Secret Key

[Erik Andersson]: This key is stored in **Secrets Manager** under the name `Delete Integration Secret Key` and must be added to the `Authorization` header when making Clear Integration requests.

> **CRITICAL SECURITY WARNING**: This key should never be leaked. If a customer obtained this key, they could theoretically delete every integration in the system, which is "suboptimal" from a security perspective. Please do not leak this key.

[Michal Rosikiewicz]: Does this key work with all kinds of integrations?

[Erik Andersson]: Yes, every integration. However, the key only authorizes the trigger call to Clear Integration. The internal uninstallation flow that follows uses the normal credentials that the customer provided during the original installation. The key is agnostic to which CRM system (Dynamics, Lime, EDL, etc.) because the system loads the relevant credentials for that integration during execution.

### Future Optimization: Delegation Keys

[Erik Andersson]: Technically, this should ideally be a delegation key for better security practices, but that's something that can be changed in the future to optimize the approach.

---

## Managing Webhook Callbacks After Deletion

### The Legal and Data Privacy Concern

When an integration is forcefully deleted on the Apsis side but the customer hasn't cleaned up their CRM system, a critical problem emerges:

[Erik Andersson]: If we fail to remove things in their CRM, they will still be trying to send webhook updates to us. From a legal perspective, this is very bad. If people's sensitive data is being sent to us after deletion, the customer (not Apsis) is technically at fault, but we don't want this situation to occur.

Currently, **the system prevents users from proceeding if there are any issues during uninstallation** for this reason.

### Proposal: Force Delete Button with Disclaimers

[Michal Rosikiewicz]: Should we add a force deletion button to the UI that allows customers to forcefully delete integrations if they're aware of what they're doing?

[Erik Andersson]: Absolutely, there's nothing technically preventing us from doing this. The button would call the normal uninstallation endpoint but append the query parameter `ignore_external_system_errors`. However, the disclaimer must state: **"Be sure to manually delete this in the CRM if you do that."**

### Monitoring for Lingering Webhook Data

After a forced deletion, it's important to verify whether the customer's CRM is still sending data:

[Michal Rosikiewicz]: We could check if they are still sending data to us after the deletion, then inform them that we've deleted it from the Apsis side but they're still sending data and need to clear their CRM system.

[Erik Andersson]: Currently, the system doesn't display the data being received from webhooks. There's technically nothing preventing us from logging it, but that would give us access to random people's sensitive data with no connection to Apsis, which presents its own privacy concerns.

---

## Step-by-Step: Deleting an Integration Using the Clear Integration Endpoint

### Required Parameters

For each deletion request, you must configure:

1. **Account ID**: The unique identifier for the customer account (note: legacy accounts may use account names instead of UIDs, e.g., "Meltex Plastics")
2. **Section ID**: The specific section within the account
3. **Integration ID**: The specific integration to delete (e.g., "lime", "fsc_enterprise_12_0")

[Erik Andersson]: These three parameters form the "holy Trinity" of pinpointing one unique installation. Without the correct integration ID, the request will fail.

> **WARNING**: The integration ID varies by system. For Lime, it's simply "lime". For FCC Enterprise 12.0, it must be entered as "FSC Enterprise 2". Entering the wrong ID will cause the request to fail, preventing accidental deletion of unintended integrations.

### Configuration in Postman

[Michal Rosikiewicz]: The Postman collection should have environment variables pre-configured for the account ID, section ID, and integration ID. The URL is already configured, so you only need to supply these three identifiers.

[Erik Andersson]: You also need to configure the **Authorization header** with the Delete Integration Secret Key for each request.

### Practical Example: Meltex Plastics Account

In the actual case discussed:

- **Account Name**: Meltex Plastics (legacy account, predated migration to UID-based account IDs)
- **Section ID**: 16134
- **Integration ID**: lime
- **Endpoint**: `https://integrations.apsis.one/accounts/Meltex%20Plastics/sections/16134/integrations/lime`

---

## Execution and Error Handling During Deletion

### Initial 401 Unauthorized Error

When the first deletion attempt was made with incorrect account name details:

- Response: `401 Unauthorized - "The provided credentials you have are not valid. Please contact support"`
- Root cause: The account ID was incorrect

### Subsequent 404 Error

- Response: `404 Not Found - "Integration was not found"`
- This indicated the endpoint was being reached but the integration wasn't found, likely due to the account ID mismatch

### Actual Execution with Correct Parameters

Once the correct account name "Meltex Plastics" and section ID were used:

1. **Initial error from Lime**: An HTML 502 Bad Gateway error was returned (customer's CRM has some misconfiguration)
2. **Apsis-side deletion**: Despite the Lime error, the `ignore_external_system_errors` flag ensured that all Apsis database entries, credentials, and configuration were deleted successfully
3. **Result**: The integration was removed from Apsis, but the webhook configurations remain in the customer's Lime instance

[Erik Andersson]: The logs show "gateway so like they they have some weird configuration here. But now we have completely disregarded this. We have deleted everything inside of Apsis."

### Known Edge Cases During Uninstallation

[Erik Andersson]: There's an annoying edge case where we have a squid proxy configuration. Very rarely, when you try to uninstall, the database refuses to delete the squid proxy entry and the request just hangs. This is one reason the `ignore_external_system_errors` flag is valuable.

---

## Post-Deletion Follow-Up: Communicating with Customers

### Required Customer Communication

After performing a forced deletion, Support must inform the customer:

> "We have forcefully deleted this integration from the Apsis side, but we detected that you are still sending us data via webhooks. Please clear your CRM system (specifically, remove any lingering webhook configurations that point to Apsis) to prevent sensitive data leakage."

[Michal Rosikiewicz]: We should check the logs to confirm they're still sending data, then provide this information to Support so they can communicate it to the customer.

### When Forced Deletion Is Necessary

[Erik Andersson]: There are specific scenarios where forced deletion is the only option:

1. **No users in the account**: We can't delete event listeners from Audience without user accounts
2. **Account has been terminated**: We can't exchange delegation keys
3. **CRM misconfiguration**: The external system throws errors but the customer still wants to delete the integration

In these cases, the `ignore_external_system_errors` flag allows us to proceed while cleaning up our side.

---

## Integration Manager Logs and Verification

### Where to Check for Successful Deletion

After executing a Clear Integration request, verification can be done in:

1. **Integration Manager**: Search for the customer account and integration to confirm it's been deleted
2. **Application logs**: The Clear Integration flow logs various steps including authorization checks and deletion operations

### Missing Logging for Failed Authorization

[Erik Andersson]: The error handling for 401 (invalid credentials) isn't fully logged in the application. When an invalid account ID is provided, the system returns a 401 error but doesn't log it, making it harder to debug. This is a known gap in the implementation.

---

## Data Storage and Architecture Notes

### Database Technology

[Michal Rosikiewicz]: I thought the integration data was stored in DynamoDB?

[Erik Andersson]: No, we use **RDS** (Relational Database Service) for integration storage, not DynamoDB. DynamoDB is not used in the Integrations domain.

---

## Customer Support Process and Issue Tracking

### Zero-Point Story Tracking Approach

[Michal Rosikiewicz]: When customer support issues are handled, the team creates a story on the board estimated at zero points, even if the issue takes only a few minutes to fix. This serves multiple purposes:

1. **Visibility**: Ensures the issue is tracked and visible to the team
2. **Historical record**: Creates a record that can be searched and referenced for similar issues in the future
3. **Knowledge building**: Allows the team to develop institutional knowledge about recurring problems
4. **Coordination**: Prevents issues from being lost or handled invisibly, especially important in larger teams

[Erik Andersson]: In smaller teams (3-4 people), a rotating schedule for checking help channels often works, with one person designated to handle support requests that take only 5 minutes. However, the zero-point story approach is more scalable.

---

## Outstanding Issues and Follow-Up Items

### Dynamics 365 Customer Inquiry

A Dynamics customer has requested information about **domain endpoints** for something related to CORS policy configuration. The exact nature of the request isn't clear from the conversation:

- Unclear whether this is about an endpoint in the CRM itself, a plugin, or the Apsis integration
- Requires clarification from the customer before proceeding

[Erik Andersson]: "I have no idea what's that. We should probably look into that together tomorrow."

### Incoming CRM Implementation Issue

A CRM implementer (Denis or similar) has reached out with issues but is attempting to bypass the proper support channels. The team needs to redirect them to official support channels while investigating whether this is an integration or audience domain issue.

### API Key Exposed in Recording

[Erik Andersson]: "I realize now that we have a recording with the API key in which is still not super [ideal]."

The Delete Integration Secret Key was inadvertently visible during screen sharing in this recorded session. This recording should be handled carefully.

---

## Key Takeaways

1. **The Clear Integration endpoint is a critical support tool** for resolving customer deletion requests when normal uninstallation fails due to external system errors, credential rotation, or misconfiguration.

2. **The Delete Integration Secret Key is highly sensitive**. It must be kept secure and never exposed in recordings, logs, or communications. It should eventually be replaced with a delegation key for better security.

3. **The "holy trinity" of parameters** (account ID, section ID, integration ID) must be correct to avoid deleting the wrong integration. Legacy accounts use account names instead of UIDs.

4. **Forced deletion on the Apsis side does not delete webhook configurations on the customer's CRM**. After deletion, customers must be explicitly informed to clean up their CRM's webhook configurations to prevent ongoing data transmission.

5. **The squid proxy edge case is a known issue** where database deletion can hang. The `ignore_external_system_errors` flag helps work around this.

6. **Zero-point story tracking for customer issues** provides better visibility, historical records, and institutional knowledge than verbal handoffs, especially in larger teams.

7. **The current implementation lacks complete error logging** for authorization failures (401 errors), making debugging harder.

---

## Unresolved Questions and Action Items

- **Customer Dynamics domain question**: Requires clarification on what specific domain endpoint information is needed and whether this is for CORS policy, plugin configuration, or integration endpoint discovery. [Erik Andersson] to follow up for more details and potentially investigate together with [Michal Rosikiewicz].

- **Error logging for 401 responses**: The error handling for invalid authorization isn't fully implemented in developer-facing logs. Should be investigated and potentially implemented.

- **Force delete button with disclaimer**: Proposed but not yet implemented. Would require UI changes and clear customer warnings about manual CRM cleanup obligations.

- **Delegation key replacement**: The Delete Integration Secret Key should eventually be replaced with a proper delegation key model for better security.

- **Recording security**: The current recording contains the Delete Integration Secret Key and should be handled accordingly (potentially redacted or restricted access).