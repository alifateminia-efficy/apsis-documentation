---
source_file: Erik - 20.11.2025.txt
domain: Apsis One Integrations
topics: [Integration Deletion Workflows, Clear Integration Endpoint, Legacy Connector Management, API Key Rotation, Webhook Cleanup, Customer Support Procedures, Debugging Tools]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Clear Integration Endpoint, Delete Integration Secret Key, Lime CRM, Efficy Enterprise, Dynamics 365, Generic Connector, Integration Manager, Postman Collections, RDS Database, Squid Proxy]
session_type: debugging-session
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session focused on troubleshooting and executing a forced deletion of a customer integration (Maltex Plastics / Lime CRM) that had failed to uninstall due to external system errors. Erik Andersson walked Michal Rosikiewicz through the **Clear Integration** endpoint—a specialized internal API that allows administrators to forcefully remove integrations by bypassing external CRM system errors. The team discussed the rationale for this restricted operation, the legal and security implications of webhook orphaning, and the proper workflow for notifying customers about lingering webhook configurations. The session also covered Postman collection setup, debugging techniques using various test endpoints, and broader process questions about issue tracking.

---

## Integration Architecture and Testing Collections

### Overview of Available Postman Collections

Erik maintains a comprehensive set of Postman collections for different connector types, which serve as both documentation and debugging tools:

- **Efficy Enterprise Collections**: Contains all relevant endpoints for the legacy Efficy Enterprise connector, demonstrating how to retrieve contacts and interact with internal APIs.
- **Generic Connector**: A full test suite allowing developers to configure target domains dynamically and verify whether integration failures stem from receiving incorrect data versus incorrect data handling.
- **Lime CRM**: Legacy connector with partial endpoint examples (not a full suite like Generic Connector).
- **Dynamics 365**: Complete endpoint documentation representing all data retrieval methods used in production. Particularly valuable for debugging customer issues involving language variations, permissions, and empty result sets.

> "If you are debugging something for a Dynamics customer, this can be very very useful because quite often we need to see like how is the data looking for Dynamics. Do they have Swedish? Do they have English? How does it look like in this language? Do we have permission? Are we getting empty result?"

---

## Clear Integration Endpoint: Purpose and Design

### What Clear Integration Does

The **Clear Integration** endpoint is a specialized administrative tool created to handle cases where normal integration uninstallation fails due to external system errors. It differs fundamentally from the standard uninstallation flow:

- **Standard Uninstallation**: Calls the normal uninstall function directly, using customer-provided credentials. If the external CRM system returns errors, the operation fails and the integration remains installed.
- **Clear Integration**: Uses an internal API key (`Delete Integration Secret Key`) to trigger uninstallation with the `ignore_external_system_errors` flag set to true, bypassing external system failures.

### Historical Context: Epic Refactoring

Two weeks prior to this session, the team completed a grooming epic to refactor the installation and uninstallation flow. The original implementation required passing flags (like `ignore_external_system_errors`) through multiple function layers, creating maintenance burden and coupling. The Clear Integration endpoint represents the solution: a dedicated endpoint that encapsulates this behavior.

### Use Case: Maltex Plastics

The specific customer case involved:

- Customer attempted to uninstall their Lime CRM integration
- Uninstallation failed because either API credentials had changed, configuration had shifted, or permissions were lost
- Customer wanted the integration deleted from Apsis despite CRM-side errors
- Solution: Use Clear Integration endpoint with ignore flag to remove all Apsis-side records (web entries, database entries, credentials) while bypassing the CRM failure

---

## Security and Authorization: Delete Integration Secret Key

### Key Storage and Access

The `Delete Integration Secret Key` is stored in the production **Secrets Manager** and must be added to the Authorization header of Clear Integration requests:

[Erik Andersson]:
> "And this is like something you would typically really not want to leak out, because in theory, if a customer had this, you can delete every integration, which is right now a bit suboptimal. So please don't leak this."

**Critical Security Note**: This key is account-agnostic. It authorizes the trigger of the uninstallation flow for *any* integration. If leaked, a malicious actor could delete all customer integrations in the system.

### How Authorization Works

- The secret key only authorizes the *trigger* of the uninstallation endpoint call
- Once triggered, the uninstallation flow internally loads the customer-provided credentials that were stored during initial installation
- Therefore, it doesn't matter which integration type is being deleted (Dynamics, Lime, Efficy, etc.)—the flow will automatically use the right credentials for that integration

---

## API Key Rotation Workflow

### Customer Option: Update API Key Without Deletion

If a customer has rotated their external CRM API key, they have two paths:

1. **Continue Using Integration**: Simply update the API key through the integration UI (e.g., for Efficy Enterprise 12.0, there is an "Update API Key" endpoint)
2. **Delete Integration and Reinstall**: Update the key first, then proceed with deletion if they no longer want the integration

[Michal Rosikiewicz]:
> "So for example they generated new key in line, would it be possible for them to update this API key? And then proceed with deletion?"

[Erik Andersson]:
> "If you just have changed your key, you don't need to uninstall at all... If you want to delete it, if you have removed the key, at least in theory you should be able to update it here and then proceed [with deletion]."

This path avoids the need for forced deletion if the CRM is still reachable.

---

## Webhook Orphaning and Legal Risk

### The Core Problem

When an integration is forcefully deleted from Apsis without successfully removing webhooks from the customer's CRM, a dangerous asymmetry emerges:

- The customer's CRM continues to send webhook notifications to Apsis endpoints
- These webhooks may contain **sensitive personal data** of the customer's contacts
- Apsis will receive these requests but reject them (no matching installation exists)
- However, the data briefly enters Apsis systems

[Erik Andersson]:
> "We will still receive the requests to these endpoints. I mean, we will of course reject them because now there exists no installation, but it's just the thing that in theory you could have access to the data if you wanted to... This is very bad from a legal perspective."

### Current Protection Measures

- Apsis **does not log the webhook payload data** itself
- Uninstallation is normally blocked if errors occur, preventing this scenario
- However, raw webhook requests are still received and could theoretically be inspected

### Why Force Delete Button is Not Yet Exposed to Customers

[Michal Rosikiewicz] advocated for adding a customer-facing "Force Delete" button with a disclaimer:

> "I don't see any value in doing it ourselves manually. If something is failing and they know that they definitely want to remove it, why not to leave that possibility for them to force the deletion?"

[Erik Andersson] acknowledged the argument but cited the legal risk:

> "The reason we are not exposing this today is because it does have a drawback because we are failing to remove things in their CRM. They will still be trying to send webhook updates to us. And this is very bad from a legal perspective."

However, both agreed there is nothing technically preventing this feature. The solution would be:
- Add a "Force Delete" button calling the same endpoint with `ignore_external_system_errors=true` query parameter
- Include a prominent disclaimer requiring the customer to manually delete webhooks in their CRM
- This matches the pattern of the standard uninstall endpoint

### Monitoring for Ongoing Webhook Traffic

After forced deletion, the team should:

1. Check **Integration Manager logs** to see if the customer's CRM continues sending webhook requests
2. Monitor the ongoing traffic pattern (it may persist for days or weeks)
3. Inform the customer: *"We forcefully deleted the integration on the Apsis side, but we are still receiving data from your CRM. Please remove any lingering webhook configurations."*

---

## Required Parameters for Clear Integration Requests

### The "Holy Trinity"

Every Clear Integration deletion request requires exactly three pieces of configuration:

1. **Account ID**: The customer's account identifier (may be legacy account name like "Maltex Plastics" rather than modern UUID format)
2. **Section ID**: The section within that account where the integration exists
3. **Integration ID**: The specific integration identifier (e.g., "Lime", "FSC Enterprise 12.0")

[Erik Andersson]:
> "Because again, this is the holy trinity of being able to pinpoint one unique installation."

These three parameters ensure that accidental deletions of unintended integrations cannot occur. The endpoint validates that all three parameters identify an existing installation before proceeding.

### Finding These Values

- **Account ID**: Must be retrieved from production systems (customer database or admin interface)
- **Section ID**: Provided by customer support team or found in Integration Manager
- **Integration ID**: Matches the integration type name exactly (e.g., "Lime" for Lime CRM, "FSC Enterprise 12.0" for Efficy)

If any parameter is incorrect, the deletion fails with a validation error rather than deleting the wrong integration.

---

## Execution: Maltex Plastics Case Study

### Debugging Initial Failures

When Michal first attempted the deletion:

1. **401 Unauthorized**: Initial attempt returned 401 error, indicating authorization header issue
2. **404 Not Found**: "Installation was not found" error after credential correction
3. **Root Cause**: Incorrect account ID provided. The customer account is named "Maltex Plastics" (with 's'), not "Maltex Plastic"

[Erik Andersson]:
> "They have just given us the incorrect account ID."

This demonstrates why the three-parameter validation is critical: incorrect parameters fail safely rather than deleting random integrations.

### The Squid Proxy Edge Case

During the actual deletion, an interesting edge case surfaced:

[Erik Andersson]:
> "Remember like this squid proxy that we have for some reason. Occasionally, very rarely, when you try to uninstall, the database refuses to delete that squid proxy entry like it just hangs on that request for whatever reason."

In this case:
- The deletion request encountered a proxy configuration issue in the customer's setup
- The error manifested as an HTML gateway error rather than a structured API error
- The ignore flag was correctly applied, and the system deleted all Apsis records anyway
- The proxy configuration will remain untouched in the CRM

### Successful Completion

Once parameters were corrected:

1. Clear Integration endpoint was called with proper account ID, section ID, integration ID, and Delete Integration Secret Key
2. System received an error from the external Lime system (proxy configuration issue)
3. **Ignore external system errors** flag suppressed this error
4. All Apsis-side records deleted successfully
5. Webhook entry points on the customer's CRM side remain active

---

## Integration Manager as Debugging Tool

### Verifying Integration Existence

Before attempting deletion, the team used **Integration Manager** in the production admin interface to:

1. Search for the customer account
2. Confirm the integration exists
3. Verify the correct integration ID and section ID
4. Check for any related configurations or data

### Log Inspection

After deletion execution, Integration Manager logs showed:
- Initial 401 error (authorization header issue)
- Subsequent 404 error (invalid account ID)
- Final deletion logs showing the external system error and its successful suppression
- Confirmation that database deletion completed

### Database Structure

Integration data is stored in **RDS** (not DynamoDB). This contains:
- Installation records for each integration
- Associated credentials (encrypted)
- Webhook configuration references
- Customer metadata

---

## Postman Collection Configuration

### Environment Variables

The Postman collection requires setup with environment variables for:

- **Authorization Header**: Must be manually added from Secrets Manager's `Delete Integration Secret Key`
- **Base URL**: `https://integrations.apsis.one`
- **Path Parameters**: Account ID, section ID, integration ID inserted per request

[Erik Andersson]:
> "Each in the collection you specify like the account section and integration ID but on each request to configure the authentication."

### Request Format

The endpoint path follows this pattern:

```
DELETE /accounts/{account_id}/sections/{section_id}/integrations/{integration_id}
Authorization: <Delete Integration Secret Key>
?ignore_external_system_errors=true
```

### Not Saving Credentials in Postman

For security, the session deliberately avoided saving the Delete Integration Secret Key as a Postman environment variable. Instead:
1. Copy the key from Secrets Manager
2. Paste into Authorization header for each request
3. Never commit to version control or save to collection

---

## Process and Workflow Improvements

### Issue Tracking for Customer Support

The team discussed how to track ad-hoc customer issues that require investigation but don't warrant full story creation:

[Michal Rosikiewicz]:
> "If we have something from customer that we have to deal with, we usually put it on board and estimate it to zero points and this is because then we can take it the next day or day after. We know that we have to deal with it and complete the story to satisfy customer. Otherwise we can have multiple issues that nobody will even know that we have."

**Benefits**:
- Creates searchable history for similar future issues
- Ensures accountability (no issues fall through cracks)
- Tracks patterns over time
- Zero-point stories distinguish support work from feature development

### Historical Context: Rotating Support Shifts

[Erik Andersson] noted that when the team was smaller (3-4 people):
> "We had a rotating schedule like who checked the help channels and then that person handles requests like this takes care of it in like 5 minutes if they notice like yeah, no, this will require like major efforts."

However, the current team has found zero-point story tracking more effective for scaling support work.

---

## Upcoming Issues and Next Steps

### Issue 1: External CRM Implementation Contact

A CRM implementation partner (Denis) has reached out with issues but is attempting to bypass proper support channels. The team needs to:
- Redirect him to proper support flows
- Triage whether the issue is integration-related or audience-related
- Investigate during the week if needed

### Issue 2: Dynamics Domain Configuration

A Dynamics customer is requesting information about domains for CORS policy configuration:

- Suspected domain: `prod-dynamics.contact.apsis.ap` or similar
- Unclear whether this is an endpoint in the CRM plugin, a webhook domain, or something else
- Requires investigation together with both Erik and Michal

[Erik Andersson]:
> "I I need to ask Eric what's for some more information, but yeah, we should probably look into that together tomorrow."

### Issue 3: Arek (Support) Notification

Michal will document the Maltex Plastics deletion in the help channel with:
- Confirmation that force deletion was successful via Clear Integration endpoint
- Instruction for Arek to notify customer to remove webhook configurations from Lime
- Any relevant error details and resolution steps

---

## Key Takeaways

1. **Clear Integration Endpoint**: A specialized administrative tool for forcefully deleting integrations when external system errors prevent normal uninstallation. Uses internal secret key authorization and ignores external system errors.

2. **The "Holy Trinity"**: Account ID, section ID, and integration ID must be correct to safely pinpoint a unique installation. Validation prevents accidental deletion of wrong integrations.

3. **Webhook Orphaning Risk**: Force-deleted integrations may continue receiving webhook traffic from the customer's CRM. This poses legal risk due to personal data exposure. Current mitigation: don't expose force delete to customers without a mandatory manual cleanup disclaimer.

4. **Secret Key Security**: The `Delete Integration Secret Key` is account-agnostic and extremely sensitive. If leaked, anyone could delete any integration. Never commit to version control or save to persistent Postman environments.

5. **Debugging with Collections**: Postman collections for each connector type (Dynamics, Lime, Efficy, Generic) are valuable for:
   - Verifying whether integration failures stem from bad data or bad handling
   - Testing connector behavior independently
   - Documenting API usage

6. **Integration Manager**: Admin interface for verifying integration existence, inspecting logs, and confirming deletion completion in RDS database.

7. **Customer Support Tracking**: Zero-point Jira stories for customer issues create searchable history, prevent lost issues, and scale support team processes.

---

## Unresolved Questions and Action Items

### Resolved During Session
- ✅ Maltex Plastics Lime integration successfully force-deleted
- ✅ Account ID mismatch identified and corrected
- ✅ Squid proxy edge case encountered and handled via ignore flag

### Pending Investigation
- **Dynamics Domain Configuration**: Identify what `prod-dynamics.contact.apsis.ap` refers to and whether customer needs it for webhook or CRM-side endpoint. Erik to gather more info; Michal and Erik to investigate together.
- **Denis (Implementation Partner) Issues**: Determine if issue is integration or audience related. Triage during the week.

### Action Items
- **Michal**: Write update in help channel for Arek with:
  - Confirmation of successful deletion via Clear Integration
  - Instruction to notify customer: "Please remove webhook configurations from your Lime CRM"
- **Erik**: Request clarification from customer on Dynamics domain question; escalate to Michal if investigation required
- **Future Enhancement**: Consider exposing Force Delete button to customers with mandatory disclaimer about webhook cleanup responsibility