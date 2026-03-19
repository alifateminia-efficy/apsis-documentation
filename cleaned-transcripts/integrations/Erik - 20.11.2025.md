---
source_file: Erik - 20.11.2025.txt
domain: Integrations
topics: [Integration Deletion Flow, Clear Integration Endpoint, API Key Management, CRM Uninstallation, Webhook Management, Postman Collections for Testing, Lime Integration, Dynamics 365 Integration, FSM Integration, Database State Management]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Clear Integration Endpoint, Delete Integration Secret Key, Postman Collections, Integration Manager, Lime CRM, Dynamics 365, FSM, RDS Database, Webhook Webhooks, Secrets Manager]
session_type: debugging-session
---

## Session Overview

This knowledge transfer session documents a live debugging and remediation scenario where the team performed a forced deletion of a customer integration (Lime CRM) that was failing to uninstall due to API credential issues. The session covers the architecture and use of the **Clear Integration endpoint**, the associated **Delete Integration Secret Key**, the rationale behind ignoring external system errors during deletion, and step-by-step execution of a manual force-deletion via Postman. The team also discusses webhook orphaning risks, database state management, and future feature improvements around user-facing force-deletion capabilities.

---

## Integration Testing Infrastructure and Postman Collections

### Overview of Available Collections

The team maintains comprehensive Postman collections for testing and debugging integrations across multiple CRM systems:

- **FCC Enterprise Collection** – Legacy connector for FCC Enterprise. Contains examples of relevant functionality used in integrations (e.g., get all contacts endpoint)
- **Generic Connector Collection** – Full endpoint coverage for the generic connector with configurable domain parameter. Useful for isolating whether issues stem from data retrieval vs. handling
- **Lime Collection** – Legacy connector; partial example coverage (not a complete test suite)
- **Dynamics 365 Collection** – Complete endpoint coverage for all data retrieval operations used in the platform

[Erik Andersson]: > "If you are debugging something for a Dynamics customer, this can be very very useful because quite often we need to see like how is the data looking for Dynamics. Do they have Swedish? Do they have English? How does it look like in this language? Do we have permission? Are we getting empty result?"

These collections serve both as debugging tools and as sources of truth for what data the platform can access from each system.

---

## Clear Integration Endpoint Architecture

### Purpose and Design

The **Clear Integration endpoint** was created as part of a previously groomed epic to refactor the installation/uninstallation flow. The core problem it solves: during normal uninstallation, parameters (especially error-handling flags) had to be passed through four or five function layers, creating fragility.

[Erik Andersson]: > "What this Clear Integration collection does is that it does call the uninstallation function but it is using the these like ignore errors flags."

### Core Use Case: Forced Deletion Due to API Credential Failure

A typical scenario: a customer's CRM system changes API credentials or misconfigures permissions. The uninstallation request fails at the external system. The customer wants to remove the integration from Apsis anyway.

The Clear Integration endpoint handles this by:
1. Accepting an internal authorization key (the **Delete Integration Secret Key**)
2. Triggering the uninstallation flow with the `ignore_external_system_errors` flag set to `true`
3. Removing all Apsis-side database entries, credentials, and web registrations **regardless of external system failures**
4. Leaving the customer responsible for manually cleaning up CRM-side artifacts (webhooks, etc.)

---

## Delete Integration Secret Key Management

### Key Storage and Security

The **Delete Integration Secret Key** is stored in AWS Secrets Manager under the key name:
```
Delete Integration Secret Key
```

This key must be retrieved from Secrets Manager and passed in the `Authorization` header of requests to the Clear Integration endpoint.

### Critical Security Implications

[Erik Andersson]: > "Because in theory, if a customer had this, you can delete every integration, which is right now a bit suboptimal. So please don't leak this."

**⚠️ WARNING**: This key grants the ability to delete **any integration in the system**. Do not expose it in code, logs, recordings, or shared documentation. Note: This transcript recording inadvertently exposed the key during demonstration—remediation (key rotation) should be prioritized.

### Key Authorization Scope

The Delete Integration Secret Key only authorizes the **trigger call** to the Clear Integration endpoint. The actual uninstallation then uses the customer's own **stored credentials** for the specific CRM system (Dynamics, Lime, FSM, etc.). Therefore:

- The key works for all integration types (Dynamics, Lime, FSM, etc.)
- The relevant customer credentials are loaded and used during the triggered uninstallation
- Deleting one integration does not affect others, because the uninstallation is tied to the specific integration ID

---

## Identifying Integrations: The "Holy Trinity"

To execute a forced deletion, three pieces of information **must** be correctly specified:

1. **Account ID** (or legacy account name) – e.g., "Maltex Plastics" (pre-UID migration accounts use the literal customer name)
2. **Section ID** – numeric identifier for the organizational section within the account
3. **Integration ID** – the system identifier for the integration (e.g., "Lime", "FSM_12.0", "Dynamics_365")

[Erik Andersson]: > "This is the holy Trinity of being able to pinpoint one unique installation."

The endpoint will fail if the Integration ID does not match the expected system identifier for that account. This is a safety measure to prevent accidental deletion of unintended integrations.

### Integration ID Format Notes

- For Lime integrations: the integration ID is simply `Lime`
- For FSM systems: the format is system-version specific (e.g., `FSM_12.0` or `FSC_Enterprise_2`)
- The system validates that the specified Integration ID exists before proceeding

---

## Request Configuration in Postman

### Required Parameters

Each request to the Clear Integration endpoint must include:

- **Account ID or Name** – Use the customer's account identifier from the Integrations Manager
- **Section ID** – The section within that account (can be retrieved from account details)
- **Integration ID** – The exact system identifier tied to that installation
- **Authorization Header** – The **Delete Integration Secret Key** from Secrets Manager

### Environment Variables

The Postman collection defines environment variables for different deployment environments, but they must be populated manually:

- `URL_prod` – Production API endpoint (e.g., `https://integrations.upsis.one`)
- `URL_apac` – APAC region endpoint
- `URL_stage` – Staging environment endpoint

**Note**: The collection may not have all environment variables pre-populated; you must add them or work with pre-configured values.

---

## The `ignore_external_system_errors` Flag and Its Implications

### What the Flag Does

When set to `true`, this flag causes the uninstallation flow to:
- Ignore HTTP errors, timeouts, and authentication failures from the external CRM system
- Proceed with removing all Apsis-side state (database entries, stored credentials, webhook registrations)
- Complete successfully even if the CRM cannot be reached or rejects the request

### Why It Exists: Valid Use Cases

1. **CRM API Credentials Rotated** – Customer changed API key in Lime/Dynamics, and Apsis cannot authenticate
2. **CRM Permissions Removed** – The user account or token no longer has uninstall permissions
3. **CRM Misconfiguration** – The CRM has a malformed configuration that prevents the uninstall endpoint from being reached (e.g., firewall, gateway error)
4. **Account Terminated** – The customer account in the CRM no longer exists
5. **No Users in Account** – Apsis cannot delete event listeners from Audience because no users exist to authenticate the request

### Critical Caveat: Webhook Orphaning

[Erik Andersson]: > "If something is failing and we have removed things in their CRM, they will still be trying to send webhook updates to us. And this is very bad from a legal perspective."

**The Major Risk**: Even after forced deletion, the customer's CRM system may continue sending webhooks to Apsis. If the CRM sends personally identifiable data (PII) for contacts/users:

- Apsis will **reject the requests** (no installation exists to route them)
- However, the webhook payload is still **technically accessible** in request logs
- From a data protection/legal perspective, this is problematic: Apsis is receiving unsolicited PII with no legitimate business relationship

**Mitigation**: After forced deletion, the customer **must be informed** to manually delete all webhook configurations in their CRM system. This is non-negotiable for compliance.

---

## User-Facing Force-Deletion Feature: Current Limitation and Future Opportunity

### Why Force-Deletion Is Not Currently Exposed to Users

[Erik Andersson]: > "The reason we are not exposing this today is because it it does have a drawback... If something is failing and they want us to delete Apps still... we will still be trying to send webhook updates to us."

The team is currently conservative: if the uninstallation fails due to external system errors, the operation blocks and returns an error. This prevents the webhook orphaning scenario.

### Proposal: Add a Disclaimer-Based Force-Delete Button

[Michal Rosikiewicz]: > "If something is failing and they know that they definitely want to remove it, why not to leave that possibility for them to force the deletion?"

The consensus is that a **user-facing force-delete button** is technically feasible and could improve user experience:

- Add a "Force Delete" button in the integration management UI with a prominent disclaimer
- The disclaimer must clearly state: *"Forcing deletion will remove this integration from Apsis, but will NOT remove webhook configurations from your CRM. You must manually remove webhooks from your CRM system to stop receiving errors."*
- Under the hood, it calls the same Clear Integration endpoint with `ignore_external_system_errors=true`

This would eliminate the need for manual Apsis support intervention in many cases.

---

## Monitoring Webhook Activity After Forced Deletion

### The Risk of Lingering Webhooks

After a forced deletion is executed, it's crucial to verify that the customer's CRM is no longer sending data to Apsis. However:

- The **Integration Manager** does not currently display the actual webhook payloads being received
- Raw logs exist, but they contain potentially sensitive customer data
- There is no built-in "webhook activity monitor" in the UI

### Proposed Monitoring Approach

If the team wants to verify that a customer's CRM is still sending webhooks post-deletion:

1. Check **request logs** for the Integration Manager service
2. Look for incoming HTTP requests to the integration endpoints from the customer's CRM
3. Observe the **request rate** and **timestamps** to determine if webhook sending is ongoing
4. Cross-reference with the deletion timestamp to confirm the customer is still sending despite deletion

[Erik Andersson]: > "We won't import it to one, but we will still receive the requests to this endpoints. I mean, we will of course reject them because now there exists no installation, but it's just the thing that in theory you could have access to the data if you wanted to."

**Privacy Consideration**: The team is appropriately cautious about not logging or storing webhook payloads, as this could inadvertently preserve customer PII.

---

## Practical Execution: Step-by-Step Walkthrough

### Scenario: Maltex Plastics – Lime Integration Deletion Failure

A customer (Maltex Plastics) is failing to uninstall their Lime integration, likely due to API key rotation or authentication issues on the Lime side.

### Step 1: Gather Integration Identifiers

Retrieve from Integration Manager or internal records:
- **Account ID/Name**: `Maltex Plastics` (legacy pre-UID account)
- **Section ID**: `16134` (numeric)
- **Integration ID**: `Lime`

### Step 2: Retrieve the Delete Integration Secret Key

Navigate to AWS Secrets Manager and copy the value of `Delete Integration Secret Key`. This will be a token/API key string.

### Step 3: Configure Postman Request

In the Postman collection:
1. Set the request method to `DELETE` (or appropriate HTTP verb for the endpoint)
2. Set the URL to: `https://integrations.upsis.one/api/v1/accounts/{account_id}/sections/{section_id}/integrations/{integration_id}`
3. Replace `{account_id}`, `{section_id}`, and `{integration_id}` with actual values:
   ```
   https://integrations.upsis.one/api/v1/accounts/Maltex%20Plastics/sections/16134/integrations/Lime
   ```
4. Add the Authorization header:
   ```
   Authorization: {value_from_secrets_manager}
   ```

### Step 4: Send the Request

Execute the DELETE request in Postman. Possible responses:

- **401 Unauthorized** – The Delete Integration Secret Key is invalid or expired
- **404 Not Found** – The Integration ID, Account ID, or Section ID is incorrect
- **500/Gateway Errors** – The backend service encountered an error (check logs)
- **200 OK with error detail in response body** – The deletion was blocked; review error message

### Step 5: Verify Deletion in Integration Manager

Search Integration Manager for the account/section/integration combination:
- The integration should no longer appear
- Database entries should be removed
- Stored credentials should be deleted

### Step 6: Check Logs for Deletion Confirmation

Examine the backend logs (e.g., via logging service) for entries indicating:
- The Clear Integration endpoint was called
- The delete_integration function was invoked
- Database deletion operations completed

[Erik Andersson]: > "We ran into an error. So like this, this, This is why we failed to. This is why you failed to delete. But now we ignored this... we have completely disregarded this. We have deleted everything inside of Apsis."

If logs show an error was encountered but deletion proceeded anyway, the `ignore_external_system_errors` flag worked as intended.

### Step 7: Notify Customer of Webhook Cleanup Requirement

[Erik Andersson]: > "We will ask SOC to ask the customer to... please inform the customer to remove any lingering web hook configuration."

Send a message to the customer (via support channel) with instructions:

> *"Your integration has been successfully removed from Apsis. However, your CRM system may still have webhook configurations pointing to our service. Please manually delete the following webhooks in your CRM to prevent errors:*
> - *[webhook endpoint URL]*
> - *[any other relevant CRM-side configuration]*
> *If you have questions, please contact our support team."*

---

## Historical Context: Why This Architecture Exists

### The Parameter-Passing Problem

The uninstallation flow originally required passing error-handling flags (like `ignore_errors`, `ignore_external_system_errors`) through multiple function layers:

```
clear_integration()
  → uninstall_integration(ignore_errors=true)
    → remove_database_entries(ignore_errors=true)
      → remove_webhooks(ignore_errors=true)
        → call_external_system(ignore_errors=true)
```

This was fragile: flags had to be threaded through 4–5 function calls, increasing the risk of missing a flag or losing context.

### The Solution: Dedicated Clear Integration Endpoint

Rather than refactoring the entire uninstall chain, the team created a dedicated endpoint specifically for forced deletion. This encapsulates the error-handling logic in one place and provides a clear API contract:

- **Input**: Account ID, Section ID, Integration ID, Authorization Key
- **Behavior**: Uninstall with all errors ignored
- **Output**: Integration deleted from Apsis; customer responsible for CRM cleanup

This is a pragmatic trade-off: a specialized endpoint is easier to audit, secure, and maintain than refactoring a deeply nested function chain.

---

## Known Issues and Quirks

### The Squid Proxy Edge Case

[Erik Andersson]: > "Occasionally, very rarely, when you try to uninstall, the database refuses to delete that squid proxy entry like it just hangs on that request for whatever reason."

During uninstallation, the system attempts to remove a squid proxy entry from the database. In rare cases, this delete operation hangs indefinitely. The issue is intermittent and not yet root-caused.

**Workaround**: Retry the deletion. If it hangs again, escalate to the database team.

### API Key Rotation and Secrets Manager

The **Delete Integration Secret Key** is a sensitive credential that should be rotated periodically (following your organization's security policies). If the key is exposed (as happened in this session's recording), initiate an immediate rotation:

1. Generate a new key
2. Update Secrets Manager with the new value
3. Notify the team of the rotation
4. Update any stored references (but ideally, don't store it—fetch from Secrets Manager each time)

### RDS vs. DynamoDB

[Erik Andersson]: > "It's it's in RDS. We don't use Dynamo DB in in integration."

The Integrations domain uses **RDS (Relational Database Service)** for persistent state, not DynamoDB. This is important when troubleshooting database issues or queries.

---

## Related Integration Scenarios

### Updating API Credentials Without Deletion

If a customer has rotated their API key but still wants to use the integration:

1. Use the "Update API Key" endpoint (e.g., `POST /integrations/{id}/credentials`)
2. Provide the new key/token from the CRM system
3. Test the connection with a small operation (e.g., get contacts)
4. Resume normal operation

No uninstallation needed.

### Testing with Postman Collections

For any integration debugging:

1. Identify which CRM system the customer uses (Lime, Dynamics 365, FSM, etc.)
2. Locate the corresponding Postman collection
3. Configure the collection with the customer's account/section/integration IDs
4. Execute requests to reproduce the issue
5. Inspect request/response payloads to isolate the problem

---

## Future Improvements and Open Questions

### User-Facing Force-Delete Button

**Status**: Agreed in principle, pending implementation.

**Requirements**:
- UI button in integration management screen
- Prominent warning/disclaimer about webhook orphaning
- Calls Clear Integration endpoint with `ignore_external_system_errors=true`
- Automatic customer notification about CRM cleanup responsibility

**Owner/Next Steps**: To be discussed with UI/product team.

### Enhanced Webhook Activity Monitoring

**Status**: Identified as useful but not implemented.

**Proposal**:
- Add a "Webhook Activity" view in Integration Manager
- Show last N webhook requests received from a deleted integration
- Display request timestamps, HTTP status, and (carefully) request metadata
- Help support teams verify that webhooks have truly stopped

**Privacy Caveat**: Must not log or display actual webhook payloads, as they may contain PII.

### Delegation Keys vs. Internal Keys

[Erik Andersson]: > "Technically, if we were to do this real really like, maybe this should be a delegation key, but this is something we slash you can change in the future to optimize this but."

The **Delete Integration Secret Key** is currently a hardcoded internal key. A more robust approach would be to use **delegation keys** (short-lived, scoped tokens) generated on-demand. This would reduce the blast radius if the key is compromised.

**Owner/Next Steps**: Architecture review with security team.

---

## Operational Workflows and Team Processes

### Customer Support Workflow for Integration Deletion

When a customer requests integration deletion due to a failure:

1. **Triage**: Determine if it's a simple credential rotation (use Update API Key) or a full deletion
2. **Attempt Normal Uninstall**: Try standard uninstall endpoint first
3. **If Failure**: Check logs to understand the root cause (missing credentials, permissions, misconfiguration)
4. **Execute Forced Deletion**: Use Clear Integration endpoint with appropriate team member authorization
5. **Notify Customer**: Inform them of successful Apsis-side deletion and ask them to verify CRM-side cleanup
6. **Follow-Up**: Optionally monitor logs for 24–48 hours to confirm no more webhooks are being sent

### Ticket Tracking for Integration Issues

[Michal Rosikiewicz]: > "If we have something from customer that we have to deal with, we usually put it on board and estimate it to zero points and this is because then we can take it the next day or day after."

The team's practice:
- Create a story/task in the project board for each customer issue
- Estimate at **zero story points** (indicating it's an ad-hoc, non-feature task)
- This allows the team to track issues that don't fit the standard development workflow
- Provides historical record for pattern analysis (e.g., recurring Dynamics credential issues)
- Helps prevent issues from being "forgotten" if the primary handler is unavailable

This is especially useful for small teams; larger teams might use a separate support ticketing system.

---

## Key Takeaways

1. **The Clear Integration Endpoint** is a specialized tool for force-deleting integrations when the CRM system is unreachable or misconfigured. It should only be used by authorized support personnel.

2. **The Delete Integration Secret Key** is a highly sensitive credential that grants deletion power over all integrations. It must never be exposed, logged, or shared. If compromised, rotate immediately.

3. **The "Holy Trinity"** (Account ID, Section ID, Integration ID) must be exactly correct to avoid accidental deletion of the wrong integration. Always verify before executing.

4. **Webhook Orphaning** is the primary risk of forced deletion. Even after Apsis-side deletion, the customer's CRM may continue sending webhooks. The customer must manually remove CRM-side webhooks; Apsis cannot do this remotely.

5. **Postman Collections** are invaluable for debugging and testing integrations across Lime, Dynamics 365, FSM, and generic connectors. They represent the official API surface the platform supports.

6. **The `ignore_external_system_errors` Flag** allows deletion to proceed despite external system failures, but this should be a last resort. Always attempt normal uninstall first.

7. **User-Facing Force-Delete Button** is a good UX improvement but requires a clear disclaimer about webhook cleanup responsibilities.

8. **RDS** is the persistence layer for the Integrations domain, not DynamoDB. Database issues should be escalated with this context.

9. **Team Processes Matter**: Tracking ad-hoc customer issues in the project board (even at zero points) prevents loss of context and builds institutional knowledge.

10. **The Squid Proxy Edge Case** is a known rare issue during deletion. If it occurs, retry; if persistent, escalate to the database team.

---

## Unresolved Questions and Action Items

### Action Items Identified in Session

1. **Key Rotation** (High Priority): The Delete Integration Secret Key was inadvertently exposed in this session's recording. Rotate the key immediately and notify the team.

2. **Webhook Monitoring Implementation** (Medium Priority): Investigate whether the team should build a webhook activity monitoring feature in Integration Manager to help support teams verify post-deletion cleanup.

3. **Delegation Key Architecture** (Medium Priority): Review whether the Delete Integration Secret Key should be replaced with delegation keys for improved security posture.

4. **Clarification on Customer Domain Requirements** (Pending): A Dynamics 365 customer has inquired about CORS policies and required domains. Requires investigation into what endpoints they need to whitelist. (Erik to follow up with additional context.)

5. **CRM Implementation Partner Issue** (Pending Investigation): A CRM implementation partner (mentioned as "Denis Moi" or similar) has reported integration issues. Determine whether this is an Integrations or Audience issue and scope the investigation.

6. **Customer Notification for Maltex Plastics**: Send the customer instructions to manually remove Lime webhooks from their CRM system to prevent orphaned requests.

### Questions for Future Discussion

- Should the Clear Integration endpoint emit audit logs to a separate audit trail system for compliance purposes?
- What is the SLA for responding to integration deletion requests?
- Should there be a "soft delete" mode that disables webhooks without removing database entries?

---

## Related Systems and Dependencies

- **Integration Manager**: UI for viewing and managing integrations; needs enhancement for webhook monitoring
- **Postman Collections**: External testing/debugging tool; maintained by the team
- **AWS Secrets Manager**: Stores the Delete Integration Secret Key; access controlled
- **RDS Database**: Persistent state for all integrations; monitored by database team
- **External CRM Systems**: Lime, Dynamics 365, FSM, etc.; have independent API and webhook delivery mechanisms
- **Audience Module**: Related but separate system; integration credential delegation may involve Audience in future

---

**Transcript cleaned and structured for RAG ingestion on 2025-11-26. Original session duration: 44m 42s.**