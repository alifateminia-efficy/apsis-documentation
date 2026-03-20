---
source_file: Erik - 20.11.2025.txt
domain: Apsis One - Integrations
topics: [Integration deletion workflows, API credentials and security, Postman collections for debugging, CRM-specific integration patterns, Error handling in uninstallation flows, Webhook cleanup and data privacy, Customer support procedures]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [FCC Enterprise connector, Generic connector, Lime connector, Dynamics 365, Clear Integration API endpoint, Integration Manager, RDS database, Secrets Manager, Delete Integration Secret Key, Squid proxy]
session_type: knowledge-transfer
---

## Session Overview

This session covers the **Clear Integration** endpoint and forced deletion workflow for integrations in Apsis One. Erik demonstrates how to handle customer scenarios where integrations need to be deleted despite API credential failures or configuration issues in the external CRM system. The team walks through a live deletion for a Lime integration customer (Maltex Plastics), discussing security implications, the role of the Delete Integration Secret Key, and critical follow-up steps to prevent orphaned webhooks from continuing to send customer data.

---

## Integration Connector Types and Debugging Tools

### FCC Enterprise and Legacy Connectors

**FCC Enterprise** is a legacy connector (not a general-purpose connector) that uses the internal APIs of FCC systems. [Erik Andersson] The Postman collection for FCC Enterprise contains all relevant functionality used in integrations—for example, endpoints to retrieve all contacts with their associated request structures.

### Generic Connector Architecture

The **Generic Connector** has a full suite of endpoints in the Postman collection. Rather than hardcoding specific systems, the generic connector allows you to configure which domain you're calling at runtime, then observe the response. [Erik Andersson] This is valuable for developers working on the generic connector because it helps isolate whether an issue stems from receiving malformed data or from incorrect handling of correctly received data.

### Lime Connector

**Lime** is also a legacy connector. The Postman collection does not have a complete suite of examples, but includes some representative examples for debugging.

### Dynamics 365 Integration Collection

The Dynamics 365 collection contains all endpoints used to retrieve data from Dynamics systems. [Erik Andersson] This collection is particularly useful for debugging customer issues because it allows you to verify how data appears in different language configurations, check permission levels, and determine whether results are empty or populated—all common questions when troubleshooting Dynamics customer integrations.

---

## Clear Integration Endpoint and Forced Deletion Workflow

### Epic Context: Refactoring Installation and Uninstallation Flows

Previously, an epic was groomed to refactor the installation and uninstallation flows because the functions were performing too much work. [Erik Andersson] The team had to pass multiple flags as parameters through four or five different function layers, which became difficult to maintain.

### What Clear Integration Does

The **Clear Integration** endpoint calls the uninstallation function with an `ignore_external_system_errors` flag set. [Erik Andersson] This is specifically designed for scenarios where uninstallation fails due to changed API credentials, configuration drift, or permission issues on the customer's CRM side, but the customer still wants the integration completely removed from Apsis.

### The Delete Integration Secret Key

The Clear Integration endpoint uses a special internal API key stored in **Secrets Manager** under the name `Delete Integration Secret Key`. [Erik Andersson] This key must be added to the authorization header of the request.

> **Security Warning:** This key must not be exposed. If a customer obtained this key, they could delete any integration in the system. [Erik Andersson] The key should be treated as a highly sensitive secret.

The key is stored in Secrets Manager and retrieved as needed, but **not** included in the Postman collection for safety reasons.

### Authorization Flow

The Delete Integration Secret Key authorizes only the trigger call to the clear integration endpoint. [Erik Andersson] Once triggered, the uninstallation flow loads and uses the **customer's original credentials** (stored during the initial installation) to perform the deletion. This means the key works for all integration types—Dynamics, Lime, EDL, or any other—because the correct credentials are loaded from the installation record.

```
Request: Clear Integration endpoint (authorized by Delete Integration Secret Key)
  → Triggers internal uninstallation function
    → Loads customer credentials for the specific system
      → Attempts CRM-side deletion
        → Ignores external system errors (if flag is set)
          → Deletes internal Apsis database entries, credentials, and configuration
```

### Difference from Normal Uninstallation

The only real difference between the Clear Integration endpoint and the normal uninstallation endpoint is that Clear Integration uses the internal Delete Integration Secret Key for authorization. [Erik Andersson] Technically, this could be optimized in the future by using a delegation key instead, but the current approach works.

---

## Addressing Customer API Key Rotation

### Updating Credentials Without Deletion

If a customer has rotated their API key in their CRM system but wants to keep the integration active, they do **not** need to uninstall. [Erik Andersson] They can simply update the API key through the integration's "Update API Key" endpoint (e.g., available for FSM to Apsis 12.0 systems).

### Update-Then-Delete Scenario

If a customer has removed their API key and wants to delete the integration, one option is to:
1. Update the integration with the new API key
2. Proceed with normal uninstallation

However, [Erik Andersson] the current system does not expose a "force delete" button on the UI, which raises the question: should customers be able to force-delete with explicit acknowledgment?

---

## Data Privacy and Webhook Cleanup Concerns

### The Webhook Orphan Problem

When using forced deletion with `ignore_external_system_errors`, the Apsis side is cleaned up completely. However, **the webhooks configured in the customer's CRM are not deleted**. [Erik Andersson] This means the customer's CRM will continue attempting to send webhook events to Apsis endpoints, and these requests will be rejected (since the installation no longer exists), but the attempt to send customer data still occurs.

### Legal and Privacy Implications

From a legal perspective, this is problematic. [Erik Andersson] Even though Apsis is not responsible if a customer sends unsolicited sensitive data, receiving and potentially having access to that data is undesirable. The team does not want to log or expose the actual data payloads being sent, because that would give random access to sensitive information about people with no relationship to Apsis.

> We will still receive the requests to these endpoints. We will reject them because the installation no longer exists, but in theory you could have access to the data if you wanted to. [Erik Andersson]

The team is "Uber paranoid" about this scenario, but it's justified from a compliance standpoint.

### Possible UI Enhancement: Force Delete Button with Disclaimer

[Michal Rosikiewicz] raised the question: Why not expose a force-delete button with a disclaimer that the customer must manually remove webhook configurations from their CRM?

[Erik Andersson] agreed there is nothing technically preventing this. The button would call the same endpoint but append the query parameter `?ignore_external_system_errors=true`. However, the current conservative approach is to force support to handle forced deletions manually, ensuring webhook cleanup is addressed beforehand or communicated to the customer afterward.

### Action for Forced Deletions

When a forced deletion is performed:
1. Execute the Clear Integration request with `ignore_external_system_errors=true`
2. Inform the customer via support channel: **"Please manually delete any lingering webhook configurations from your CRM system"**
3. Verify in logs whether the customer is still sending data post-deletion

---

## Checking for Orphaned Webhook Traffic

### Detecting Continued Data Transmission

After a forced deletion, you can check the application logs to see if the customer is still sending webhook events. [Erik Andersson] The logs show request timestamps and will continue to update as long as the customer's CRM is configured to send webhooks. The log entries in the session showed data being sent as recently as "today" even though the deletion happened "13th"—indicating the customer's webhook configuration was still active.

### Integration Manager and Log Inspection

Use **Integration Manager** in production to search for the customer (e.g., by account name like "Maltex Plastics") and verify the installation record. Then check application logs to confirm whether webhook traffic has ceased post-deletion.

---

## Postman Collection Setup and Request Parameters

### Required Configuration

For each Clear Integration request in Postman, you must specify three pieces of information—the **"Holy Trinity"** of pinpointing a unique installation: [Erik Andersson]

1. **Account ID** (or account name for older customers predating UID migration; e.g., "Maltex Plastics")
2. **Section ID**
3. **Integration ID** (e.g., "Lime", "FSC Enterprise 12.0", "Dynamics 365")

### Environment Variables and Authentication

The Postman collection includes variables for different environments (APAC, Stage, Prod), but URL endpoints do not need to be edited per request. [Erik Andersson] What **must** be configured on each request is the authorization header with the Delete Integration Secret Key.

Example request structure:
```
Authorization: Bearer <Delete Integration Secret Key from Secrets Manager>
URL: https://integrations.apsis.one/accounts/{account_id}/sections/{section_id}/integrations/{integration_id}
```

### Integration ID Format

The integration ID must match the exact format stored in the system. [Erik Andersson] For Lime, it's "Lime". For FSC Enterprise versions, it must include the version (e.g., "FSC Enterprise 12.0"), not just the name. If the wrong integration ID is provided, the request will fail because the system validates that the integration ID exists before deletion.

---

## Live Deletion Example: Maltex Plastics - Lime Integration

### Problem Statement

A customer (Maltex Plastics) has a Lime integration that is failing to uninstall due to changed API credentials or configuration issues on the Lime side. The customer wants the integration completely removed from Apsis. [Erik Andersson] This is a prime use case for the Clear Integration endpoint.

### Required Information Gathering

Before executing the deletion:
- **Account Name/ID:** "Maltex Plastics" (Note: the team initially used "Meltx Plastics" by mistake, which is why the first request failed)
- **Section ID:** 16134 (confirmed via Integration Manager)
- **Integration ID:** "Lime"
- **Authorization:** Delete Integration Secret Key from Secrets Manager (value: [prod endpoint details])

### Execution Steps

1. **Retrieve the Delete Integration Secret Key** from Secrets Manager in the production environment
2. **Configure Postman request** with account, section, and integration ID
3. **Set Authorization header** with the secret key
4. **Send the request** to the Clear Integration endpoint

### Observed Error and Root Cause

The first request returned a **401 Unauthorized** error, which was initially confusing because the request appeared properly formatted. Investigation revealed the issue: the **account ID/name was incorrect** ("Meltx Plastics" instead of "Maltex Plastics"). 

[Erik Andersson] noted that when the account name is wrong, the system does not log the error; it simply returns a 401 without visibility. This is a logging gap that could be improved in future iterations.

### Edge Case: Squid Proxy Deletion Hang

During the actual deletion, the system encountered a rare edge case: [Erik Andersson] a Squid proxy configuration entry in the database that refused to delete. The request "hangs" on this entry for unknown reasons. However, with `ignore_external_system_errors=true`, this error was disregarded, and the deletion proceeded successfully.

The logs showed:
```
[Error] HTML gateway error received from Lime
[Status] Ignoring external system error - proceeding with deletion
[Result] Successfully deleted all Apsis database entries and credentials
```

### Completion and Follow-up

After the forced deletion completed:
- **Apsis side:** Completely cleaned up (database entries, credentials, configuration removed)
- **Lime side:** Webhooks still exist and will continue sending data until the customer manually removes them
- **Required action:** Support must contact the customer with: **"Please delete the webhook configuration from your Lime instance to prevent orphaned data transmission"**

---

## Failure Modes and When to Use Ignore Flags

The system currently has two flavors of "ignore" flags for uninstallation:

1. **`ignore_external_system_errors`** – Used when the CRM has misconfiguration, API credential issues, or other problems that prevent successful uninstallation on the CRM side. [Erik Andersson]

2. **`ignore_audience_errors`** (implied) – Used when there are no users in the Apsis account (can't delete event listeners), or the account has been terminated (can't exchange delegation keys).

These flags allow forced deletion when normal uninstallation cannot proceed, at the cost of potential orphaned webhook configurations that must be manually cleaned up.

---

## Support Workflow and Issue Tracking

### Zero-Point Story Approach

When a customer support issue is identified that does not immediately require a major development effort, the team creates a **zero-point story** on the board. [Michal Rosikiewicz] This ensures:
- The issue is tracked and visible to the team
- A rotating schedule determines who handles it
- Completion of the story documents resolution
- Historical record allows searching for similar issues

This approach differs from teams with fewer members (3-4 people) who may rotate a help-channel duty without formal story creation, but scales better as team size grows.

### Help Channel Routing

Customer requests often come through help channels. [Erik Andersson] The team tries to route requests through proper channels, but sometimes CRM implementers (like the person mentioned, Denis) try to bypass support and reach developers directly. The proper procedure is to go through the support/SOC team.

### Investigation-Driven Stories

If an investigation reveals the issue will take over an hour, a proper story is created and prioritized in the normal development workflow. Otherwise, it's handled as a quick fix using the zero-point story approach. [Michal Rosikiewicz]

---

## Unresolved Issues and Follow-up Items

### Domain Routing Configuration Request

A Dynamics 365 customer is asking about domains that Apsis can be reached on. They want to configure CORS policies. [Michal Rosikiewicz] The team does not yet understand the full scope of this request:
- Is it an endpoint inside the CRM plugin?
- Is it related to webhook callback domains?
- What exactly does "domains we can be reached" mean in this context?

**Action:** Erik will ask for more information and discuss with Michal tomorrow morning. If it's a straightforward question, Erik will handle it; if investigation is needed, both will collaborate.

### CRM Implementation Consultant Issue

Denis or another CRM implementation consultant has reached out with unspecified issues. [Erik Andersson] The team is trying to get them to go through proper channels. The issue may or may not be integration-related. Investigation is pending during the week.

**Action:** Team will investigate if it's not resolved through proper channels, and determine whether it's an integration or Audience-related issue.

---

## Key Takeaways

1. **Clear Integration endpoint** is the mechanism for forced deletion when CRM-side uninstallation fails. It uses a secret key and ignores external system errors.

2. **Delete Integration Secret Key** is a highly sensitive credential stored in Secrets Manager. Never expose it in Postman collections or logs.

3. **The Holy Trinity** (Account ID, Section ID, Integration ID) uniquely identifies an installation and must be correct for any deletion request.

4. **Webhook orphan problem:** Forced deletions clean Apsis but leave CRM-side webhooks active. Support must always contact the customer to request manual webhook cleanup.

5. **Data privacy concern:** Even though rejected, orphaned webhooks still transmit customer data. The team intentionally avoids logging payloads to prevent accidental exposure of sensitive information.

6. **Postman collections** for each connector system (FCC, Generic, Lime, Dynamics) are valuable debugging tools. Use them to verify data flow and isolate issues.

7. **Squid proxy edge case:** Rare deletion hangs occur; the `ignore_external_system_errors` flag handles these gracefully.

8. **Zero-point story workflow** tracks support issues without formal estimation, scales as team grows, and builds historical knowledge.

9. **Account name vs. ID:** Pre-UID customers use account names (e.g., "Maltex Plastics"). Ensure the correct name/ID is used or requests fail silently with poor logging.

10. **Future enhancement:** A force-delete button with customer disclaimer could be added to the UI; nothing technically prevents it, but conservative approach prioritizes data safety.

---

## Unresolved Questions

1. **CORS policy domain request:** What exactly does the Dynamics customer need regarding "domains we can be reached"? Is this webhook callback URLs, plugin endpoints, or something else?

2. **CRM implementation consultant issue:** What is the actual technical problem reported by Denis? Is it integration or Audience-related?

3. **Logging gap for invalid account IDs:** Should the Clear Integration endpoint log 401 errors when account ID/name doesn't exist, to improve debugging visibility?

4. **Force-delete UI button:** Should a force-delete button with a GDPR/data privacy disclaimer be implemented to allow customers self-service forced deletion?