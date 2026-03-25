---
source_file: Erik - 20.11.2025.txt
domain: Apsis One Integrations
topics: [Integration Deletion, Force Uninstallation, API Key Management, Postman Collections, Credentials Handling, CRM Configuration Issues, Webhook Cleanup]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Clear Integration Endpoint, Delete Integration Secret Key, Lime CRM, Efficy Enterprise (FSM/FSC), Dynamics 365, Integration Manager, Postman Collections, RDS Database, Squid Proxy]
session_type: debugging-session
subdomains: [Architecture, Generic Connector, Outbound Flow, Lime, Efficy Enterprise 12.0, Duplicate profiles in Apsis]
---

## Session Overview

This debugging session focused on performing a forced deletion of a Lime CRM integration for a customer (Maltex Plastics) whose API credentials had become invalid. Erik Andersson explained the architecture and mechanics of the **Clear Integration** endpoint—a specialized system that uses an internal secret key to force-delete integrations without requiring valid external system credentials. The session covered Postman collection usage, credential management, debugging procedures, and critical considerations around webhook cleanup and data privacy when forcibly removing integrations.

---

## Integration Deletion Architecture

### The Clear Integration Endpoint

The **Clear Integration endpoint** is a specialized tool designed for scenarios where normal uninstallation fails due to changed credentials, terminated accounts, or CRM misconfiguration. Unlike the standard uninstallation endpoint, it uses an internal administrative key rather than requiring customer credentials.

[Erik Andersson]: The primary difference between the Clear Integration endpoint and the normal installation endpoint is that this endpoint uses one of our internal keys, so we have one API key we can utilize to clear out existing integrations without needing to access customer accounts.

> Technically, this should probably be a delegation key for better security, but this is something that can be changed in the future to optimize this.

### The Delete Integration Secret Key

This sensitive credential must be kept secure and is stored in the **Secrets Manager** under the key name: `Delete Integration Secret Key`

[Erik Andersson]: This is like something you would typically really not want to leak out, because in theory, if a customer had this, you can delete every integration, which is right now a bit suboptimal. So please don't leak this.

**Usage**: The key is copied from Secrets Manager and added to the Authorization header of the Clear Integration API request.

### How the Forced Deletion Works

When the Clear Integration endpoint is triggered, the following flow occurs:

1. The request is authorized using the Delete Integration Secret Key
2. The endpoint calls the internal uninstallation function
3. Two special flags are used during uninstallation:
   - `ignore_errors` - to bypass external system API failures
   - `ignore_external_system_errors` - specifically for CRM system errors
4. Internal database entries are deleted regardless of external system response
5. Credentials stored in Apsis are removed

[Erik Andersson]: The ignore external system errors flag allows us to proceed with the flow even if the CRM system is throwing an error. We will still remove everything we have in our database—the web entries, database entries, and credentials. They will be able to reinstall if they want to.

### Integration-Agnostic Design

The Delete Integration Secret Key works with **all integration types** (Dynamics, Lime, Efficy, etc.) because the endpoint loads the relevant customer credentials for each system during the uninstallation process.

[Erik Andersson]: It doesn't matter which integration you are trying to do this for. Like it can be Dynamics or Lime or E-deal because in the end we will load the relevant credentials for the system.

---

## When to Use Force Deletion vs. Standard Uninstallation

### Standard Uninstallation Path

For customers who have only rotated their API keys (but still have valid credentials), use the **Update API Key** endpoint to provide new credentials, then proceed with normal uninstallation. This avoids forcing deletion and maintains clean CRM-side cleanup.

[Michal Rosikiewicz]: If you just have changed your key, you don't need to uninstall at all. You can just update it here and then proceed with deletion if desired.

### Force Deletion Scenarios

Use the Clear Integration endpoint when:

- CRM API credentials have been invalidated or cannot be updated
- The customer account has been terminated
- The CRM system has misconfiguration preventing webhook deletion
- Normal uninstallation is failing despite correct credentials

---

## Critical Data Privacy and Legal Considerations

### The Webhook Persistence Problem

**This is the primary reason forced deletion is not exposed to customers as a self-service button.**

When an integration is forcibly deleted from Apsis without successfully deleting webhooks in the external CRM system, the customer's CRM continues sending webhook requests to Apsis endpoints. While these requests are rejected (because the installation no longer exists), this creates a problematic situation:

1. The external system may be sending personally identifiable information (PII) to Apsis endpoints
2. Apsis could theoretically access this data even though the integration was removed
3. From a legal/GDPR perspective, this is highly undesirable

[Erik Andersson]: This is very bad from a legal perspective. I mean, sure, for Apsis, this kind of doesn't matter because it's not our fault that someone is sending us random person sensitive data. It's not us that would be get in trouble, but we don't want that to happen.

### Current Safeguards

- Webhook request data is **not logged** in Apsis systems
- The endpoint rejects requests if no matching installation exists
- Customers are required to notify Apsis when forced deletion is needed

### Potential Future Enhancement

A "Force Delete" UI button could be added with clear disclaimers, but would require the customer to manually remove webhook configurations from their CRM system.

[Erik Andersson]: There is absolutely nothing preventing us from adding a force delete button with a disclaimer like "be sure to manually delete these things in the CRM if you do this," but there is absolutely nothing that stops us from doing it.

---

## Postman Collection Reference Architecture

### Available Collections by Integration Type

The Postman collections provided contain example endpoints and requests for all major integration types:

- **Efficy Enterprise (FSC Enterprise)**: Full suite of endpoints used internally
- **Generic Connector**: Complete set of configurable endpoints for testing custom integrations
- **Lime CRM**: Partial example set (legacy connector, not generic)
- **Dynamics 365**: All endpoints used for retrieving customer data
- **Clear Integration Collection**: Specialized for forced deletion operations

[Erik Andersson]: These collections are very useful if you're developing something in the generic connector and you can verify—is the issue because I am getting incorrect data or is it because we are receiving it but then handling it incorrectly?

### The Debugging Collection Use Case

These are particularly valuable for customer support debugging:

> For Dynamics customers, you can verify how data looks in different languages, check permissions, verify empty results, and inspect the actual API responses being received.

---

## Performing a Forced Integration Deletion

### Required Information (The "Holy Trinity")

Every Clear Integration deletion request requires three pieces of information to pinpoint the exact installation:

1. **Account ID** (or Account Name for legacy pre-UID customers)
2. **Section ID**
3. **Integration ID**

[Erik Andersson]: This is the holy trinity of being able to pinpoint one unique installation. If you miss any of these, the request will fail.

### Postman Configuration

The Postman collection requires:

```
URL: https://integrations.apsis.one/accounts/{accountID}/sections/{sectionID}/integrations/{integrationID}
Authorization Header: [Delete Integration Secret Key from Secrets Manager]
Method: POST (or DELETE, depending on endpoint)
```

Variables needed for each request:
- `account_id` or `account_name` (for legacy accounts)
- `section_id`
- `integration_id`

[Michal Rosikiewicz]: The integration ID is not like the integration name. For Lime it's the same, but for Efficy Enterprise 12.0, you need to enter "Efficy Enterprise 12.0" exactly.

### Critical Implementation Detail

The system validates that the integration ID exists before proceeding. This prevents accidental deletion of unrelated integrations.

[Erik Andersson]: We make sure that the integration ID exists—you would not be able to completely destroy anything. The request would fail otherwise.

---

## Debugging Failed Uninstallations

### Common Failure Points

#### Invalid Credentials (401 Errors)

Occurs when the provided API credentials for the external system are no longer valid. This is the primary reason customers contact support.

[Michal Rosikiewicz]: The provided credentials you have are not valid. Please contact support.

#### Integration Not Found (404 Errors)

Results from:
- Incorrect account ID (especially critical with legacy account names vs. new UUIDs)
- Incorrect integration ID
- Integration already deleted

#### HTML Gateway Errors (5xx responses)

May indicate misconfiguration in the CRM system or proxy issues. These errors trigger the forced deletion path.

### The Squid Proxy Edge Case

[Erik Andersson]: Occasionally, very rarely, when you try to uninstall, the database refuses to delete the squid proxy entry—it just hangs on that request for whatever reason.

When this occurs:
- The uninstallation appears to hang
- Using `ignore_external_system_errors` flag allows deletion to proceed
- The squid proxy entry remains in the external system (will need manual cleanup)

### Verification After Force Deletion

Check Integration Manager logs to confirm:
1. Installation was found and deletion was initiated
2. External system errors were encountered (expected)
3. Errors were ignored (only when using force deletion)
4. All Apsis-side cleanup completed successfully

---

## Post-Deletion Customer Communication

### Required Actions

After forcing a deletion, the support team must **immediately notify the customer** to:

1. **Remove webhook configurations** from their CRM system
2. Verify that no further data is being sent to Apsis

[Erik Andersson]: Now we just need to ask support to ask the customer to do the webhook deletion. Everything is done in Apsis, but we have not deleted the webhooks in their CRM.

### Verification Workflow

Before sending deletion confirmation to customer:

1. Check if customer is still sending webhook requests to Apsis
2. If yes, provide evidence in communication: "We see you are still sending us data as of [timestamp]"
3. Instruct customer to identify and remove the webhook configuration in their CRM
4. Confirm removal once data flow stops

[Michal Rosikiewicz]: We can say OK, we forced deleted this from Apsis side, but we can see that you are still sending us data. Please clear your CRM system.

### Why This Matters

The continued webhook sends aren't breaking, but represent a data leakage risk and poor user experience.

---

## Session Walkthrough: Maltex Plastics Lime Integration Deletion

### The Problem

Customer account: **Maltex Plastics** (legacy account name, pre-UUID era)
CRM System: **Lime**
Integration ID: **lime**
Issue: API credentials invalidated; customer cannot proceed with normal uninstallation

### Initial Attempt Failure

First deletion attempt failed with:
- Error: Account not found
- Root cause: Account name was provided as "Meltx Plastics" instead of "Maltex Plastics"

[Erik Andersson]: They have just given us the incorrect account ID.

### Corrected Execution

Once correct account ID was provided:

```
Account ID: Maltex Plastics
Section ID: 111613416134 (verified in Integration Manager)
Integration ID: lime
```

Request succeeded through the following steps:

1. Authorization check passed (Delete Integration Secret Key valid)
2. Integration found in database
3. Uninstallation flow initiated with `ignore_external_system_errors` flag
4. External system returned HTML gateway error (Lime API misconfiguration)
5. Error was ignored; deletion proceeded
6. All Apsis database entries deleted successfully
7. Credentials removed from Apsis

[Erik Andersson]: Because you remember like this squid proxy that we have for some reason—occasionally, very rarely, when you try to uninstall, the database refuses to delete that squid proxy entry. But now we're done.

### Post-Deletion Status

- ✅ Apsis-side: Complete deletion
- ⚠️ Lime-side: Webhooks still configured (requires customer action)
- 📝 Next step: Notify customer to remove webhook configurations from Lime

---

## Integration Deletion Safeguards

### Why Full Access Control Matters

The system is designed with multiple layers of validation:

1. **Integration ID validation**: Prevents deleting wrong integration
2. **Account/Section validation**: Ensures isolation between customers
3. **Secret key rotation capability**: Can be changed if compromised
4. **Audit trail**: Deletion operations are logged

[Erik Andersson]: We make sure we have like this integration ID exists—you would not be able to completely destroy anything.

### Future Security Considerations

Currently, the Delete Integration Secret Key is a service-level credential. For better security practices:

- Should be converted to a **delegation key** with time-limited access
- Could be scoped to specific account deletion operations
- Rotation schedule should be implemented

[Erik Andersson]: Technically, if we were to do this really like, maybe this should be a delegation key, but this is something you can change in the future to optimize this.

---

## Postman Collection Variable Setup

### Production Environment Variables (Minimal Set)

For executing Clear Integration deletions, only these environment variables are required:

```
- account_id: [Customer account name or UUID]
- section_id: [Section number from Integration Manager]
- integration_id: [Integration type: "lime", "dynamics", etc.]
- auth_key: [Delete Integration Secret Key from Secrets Manager]
```

**Stage/APAC variables**: Not needed for production customer deletions; can be omitted.

### Collection Structure Notes

- URLs are static in collection; do not need per-request modification
- Account ID, Section ID, Integration ID **must** be configured per request
- Authentication header must be set on each request (not saved persistently)
- Postman environment variables can be used but are not strictly required

---

## Broader Postman Collection Utility

### Development and Testing

The Postman collections serve multiple purposes beyond deletion:

1. **Generic Connector Development**: Test new integration endpoints by configuring domain and inspecting results
2. **Customer Debugging**: Use Dynamics/Lime/Efficy collections to verify API responses in customer environments
3. **Language/Permission Verification**: Confirm data formatting in different languages and verify permission levels

[Erik Andersson]: If you're developing something in the generic connector and you can verify—is the issue because I am getting incorrect data or is it because we are receiving it but then handling it incorrectly?

### Collections as Documentation

Each collection serves as executable documentation of:
- All endpoints the system uses for that integration type
- Expected request/response structures
- Authentication requirements
- Parameter specifications

---

## Team Process: Customer Issue Tracking

### Approach for Investigation-Level Issues

When a customer issue is reported that requires support investigation:

1. **Zero-point estimation**: Create a story/task with 0 story points
2. **Enable tracking**: Ensures the issue doesn't get lost if multiple team members are handling support rotation
3. **Documentation**: Builds institutional knowledge searchable by integration type, customer, or issue category
4. **Historical reference**: Future similar issues can reference previous solutions

[Michal Rosikiewicz]: If we have something from customer that we have to deal with, we usually put it on board and estimate it to zero points. This is because then we can take it the next day or day after, and we know that we have to deal with it and complete the story to satisfy customer. Otherwise we can have multiple issues that nobody will even know that we have.

### Small vs. Large Teams

The approach differs based on team size:

**Small teams (3-4 people)**: Rotating support duty with quick verbal handoff may be sufficient
**Larger teams (5+ people)**: Formal issue board tracking becomes essential for visibility

[Erik Andersson]: We were three people or four people at most. So we had a rotating schedule like who checked the help channels and then that person handles requests like this in like 5 minutes.

### Historical Value

Issue board entries provide searchable history useful for:
- Finding similar past issues for faster resolution
- Building knowledge base of integration-specific problems
- Identifying patterns in particular customers or CRM systems

---

## Unresolved Questions and Follow-ups

### Domain Configuration for Dynamics

[Michal Rosikiewicz]: A customer wants to know the domains that we can be reached at. They want to put some CORS policy.

**Status**: Unclear what endpoint or configuration is being requested
**Action**: Erik to request clarification from the customer (possibly through Eric or another CRM implementer)
**Next step**: Discuss with Michal tomorrow morning if investigation required

[Erik Andersson]: I have no idea what's that. I don't know if this has some endpoint inside the CRM or the plugin or what.

### Potential Integration-Related Issue from CRM Implementer

A CRM implementation contact (Denis or similar) has reported unspecified issues but has been attempting to sidestep support processes.

**Status**: Unclear if issue is integration-related or audience-related
**Action**: Will investigate during the week; may require deeper analysis
**Next step**: Erik will attempt to direct requester through proper support channels

[Erik Andersson]: He has reached out and told me there are some issues, but I'm trying to get him to go with the proper flows. But we will most likely need to check on it like during the week.

---

## Key Takeaways

1. **Force Deletion Architecture**: The Clear Integration endpoint with the Delete Integration Secret Key is the solution for removing integrations when standard uninstallation fails due to invalid credentials or CRM misconfiguration.

2. **The Holy Trinity**: Always verify Account ID, Section ID, and Integration ID before executing a deletion to prevent accidental removal of wrong installations.

3. **Webhook Cleanup is Manual**: Forced deletion removes Apsis-side configuration only. Customers must manually remove webhook configurations from their CRM to prevent continued (though harmless) webhook requests.

4. **Data Privacy Risk**: Webhook requests from deleted integrations represent a potential data leakage vector. This is why force deletion is not self-service and requires explicit notification to customers.

5. **Integration-Agnostic**: The Delete Integration Secret Key works across all integration types by loading the appropriate customer credentials during the uninstallation flow.

6. **Debugging with Collections**: Postman collections for each integration type are valuable tools for verifying data formatting, permissions, and API response structures in customer environments.

7. **Issue Tracking for Visibility**: Even zero-point customer issues should be tracked on the board to prevent loss of visibility in larger teams and to build searchable historical knowledge.

8. **Account ID Gotcha**: Legacy customer accounts (pre-UUID) use account names instead of UUIDs. Verify the exact spelling in Integration Manager before execution.

9. **Squid Proxy Edge Case**: Occasionally uninstallation hangs on squid proxy entry deletion; the ignore_external_system_errors flag allows the overall deletion to complete despite this edge case.

10. **Security Consideration**: The Delete Integration Secret Key should eventually be converted to a delegation key with time-limited access for better security practices.

---

## Action Items

- **Michal Rosikiewicz**: Write support response to customer detailing forced deletion completion and requesting manual webhook removal from Lime CRM
- **Erik Andersson**: Request clarification from customer (via Eric or implementer) on the Dynamics domain/CORS policy requirement
- **Erik Andersson**: Investigate unspecified issue from CRM implementer, determine if integration or audience related
- **Team**: Review and discuss tomorrow morning if the Dynamics domain requirement needs deeper investigation
- **Future Task**: Consider converting Delete Integration Secret Key to a delegation key with time-limited scope for enhanced security