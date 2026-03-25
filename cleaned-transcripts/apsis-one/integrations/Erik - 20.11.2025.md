---
source_file: Erik - 20.11.2025.txt
domain: Apsis One Integrations
topics: [Integration Deletion and Force Uninstall, Postman Collection for Testing, API Key Management, Error Handling in Uninstallation Flows, Webhook Cleanup Post-Deletion, Lime Integration Troubleshooting, Customer Support Procedures]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Postman Collections, Clear Integration Endpoint, Secrets Manager, Integration Manager, Lime Integration, Efficy Enterprise Integration, Dynamics 365 Integration, RDS Database, Delete Integration Secret Key, Squid Proxy]
session_type: debugging-session
subdomains: [Architecture, Tribe Integration]
---

## Session Overview

This session focused on troubleshooting and executing a forced deletion of a Lime integration for a customer (Maltex Plastics) using Apsis One's internal tooling. The team walked through the **Clear Integration** endpoint, which allows deletion of integrations even when external system credentials have become invalid. The session covered the rationale behind the forced deletion mechanism, security considerations around the deletion API key, and the practical steps taken to complete the deletion despite encountering authentication errors from the Lime CRM system. Key discussion points included webhook cleanup responsibilities and future improvements to the UI for user-initiated force deletion.

---

## Overview of Integration Testing Collections

Erik explained the various Postman collections available for testing and debugging different integration types:

### FCC Enterprise (Legacy Connector)
[Erik Andersson]: FCC Enterprise is a legacy connector, not a general connector. The collection contains relevant functionality used in the integration, such as endpoints to retrieve all contacts. This collection is useful for verifying which data is being requested and received.

### Generic Connector Collection
The generic connector has a full suite of endpoints available. Each endpoint can be configured to call different domains, making it useful for development and verification. The value of this collection is in isolating where issues occur:

> If you're developing something in the generic connector, you can verify: is the issue because I am getting bad data, or is it because we are receiving data but handling it incorrectly?

### Lime Integration (Legacy Connector)
While Lime is a legacy connector without a complete endpoint suite, the collection includes examples for common operations.

### Dynamics 365 Collection
This collection contains all endpoints used for retrieving data from Dynamics. It is particularly valuable for debugging customer-specific issues:

[Erik Andersson]: If you are debugging something for a Dynamics customer, this can be very useful because quite often we need to see how the data is looking for Dynamics. Do they have Swedish? Do they have English? How does it look in this language? Do we have permission? Are we getting empty result?

---

## The Clear Integration Endpoint: Force Deletion Mechanism

### Background: The Refactoring Epic
[Erik Andersson]: Previously we groomed an epic where we wanted to refactor the installation and uninstallation flow because the functions are doing too much stuff today. We had issues because we had to pass flags in too many different layers — four or five different functions where we had to pass the flags as parameters everywhere.

The **Clear Integration endpoint** solves this by calling the uninstallation function with an `ignore errors` flag set, allowing deletion to proceed even when external systems fail.

### Use Case: Changed API Credentials
A customer (Lime in this case) attempted to uninstall their integration. However, their API credentials had changed or been rotated. The normal uninstallation flow failed because it could not reach the Lime CRM system to remove webhooks and clean up configuration.

[Erik Andersson]: What is actually happening now for this Lime customer? They're trying to uninstall, but for some reason something has changed — either with the API credentials, with their configuration, or our permission. So we are failing to do the uninstallation request, but they want us to delete the app still. This is a prime example where you would use the forced uninstallation because here we are setting the `ignore external system errors` flag.

### How It Works
The Clear Integration endpoint differs from the normal uninstallation endpoint in one critical way: it uses an internal API key stored in Secrets Manager, rather than requiring access to the customer's account.

[Erik Andersson]: This endpoint is using one of these internal keys that we have — one API key we can utilize to clear out existing integrations without needing to access customers' accounts. Technically, if we were to do this really carefully, maybe this should be a delegation key, but this is something you can change in the future to optimize this.

### The Delete Integration Secret Key

The secret key is stored in **Secrets Manager** under the name `Delete Integration Secret Key`.

**Critical Security Warning**: This key should never be leaked. In theory, if a customer had this key, they could delete every integration in the system, which is suboptimal.

[Erik Andersson]: The key is stored in Secrets Manager. You would copy-paste the secret and add it here in the authorization header.

### Authorization Scope
[Michal Rosikiewicz]: This key will work with all kinds of integrations?

[Erik Andersson]: Every integration. Because what you are doing here is you are triggering the uninstallation flow. During the uninstallation flow, we are utilizing the normal credentials for the installation. But this key is only the authorization for the trigger call. This request will in turn trigger the internal uninstallation which utilizes the credentials that the customer had provided during the installation.

So the key doesn't matter which integration you're trying to delete:

> It can be Dynamics or Lime or E-deal because in the end we will load the relevant credentials for the system.

---

## Alternative: Updating API Keys Before Deletion

An important point emerged during discussion: customers do not always need force deletion. If they have simply rotated their API key, they can update it through the normal UI.

[Erik Andersson]: If you just have changed your key, you don't need to uninstall at all. You can just update it here.

For systems like FSM (Efficy) 12.0, an "Update API Key" endpoint exists that allows credentials to be refreshed before normal uninstallation is attempted. If deletion is still desired after updating credentials, the standard uninstall flow can proceed.

---

## Proposed Feature: Force Deletion Button in UI

[Michal Rosikiewicz]: Can we also add a button to force the deletion on the app's side if they are fully aware of what they are doing? Because I don't see any value in doing it ourselves manually. If something is failing and they know that they definitely want to remove it, why not leave that possibility for them to force the deletion?

### Current Limitation: Webhook Cleanup Responsibility

[Erik Andersson] initially explained why this feature is not exposed today:

> The reason we are not exposing this today is because it has a drawback. If we are failing to remove things in their CRM, they will still be trying to send webhook updates to us. And this is very bad from a legal perspective.

However, Erik acknowledged that adding a force delete button with proper disclaimers is technically feasible:

[Erik Andersson]: I agree there is absolutely nothing preventing us from adding a force delete button with a disclaimer like "Be sure to manually delete these things in the CRM if you do that." There is absolutely nothing that stops us from doing it.

The implementation would be straightforward:

> This button would call the normal uninstallation endpoint, but you just append the query parameter `ignore_external_system_errors`.

---

## Data Privacy Concern: Undeleted Webhooks Sending Data

A critical issue was identified: if an integration is force-deleted on the Apsis side but the customer doesn't remove webhooks from their CRM, the CRM will continue sending data to Apsis endpoints.

[Erik Andersson]: We won't import it to Apsis One, but we will still receive the requests to these endpoints. We will reject them because the installation no longer exists, but in theory you could have access to the data if you wanted to.

[Michal Rosikiewicz]: So if we deleted this installation, we won't be proceeding with this data to import it, yeah?

[Erik Andersson]: We won't import it, but we will still receive the requests. It's just that in theory you could have access to data that has no connection to Apsis at all.

**Current Safeguard**: The system does not log the actual payload data being received, so access to customer data is not currently exposed. However, this is a concern that motivated the team to include careful customer communication about webhook cleanup.

---

## Practical Execution: Deleting Maltex Plastics Lime Integration

### Required Information: The "Holy Trinity"
To execute any deletion, three pieces of information must be configured:

```
- Account ID (or account name for legacy accounts)
- Section ID
- Integration ID
```

[Erik Andersson]: The URLs you don't need to bother about. But what you do need to add or configure every time is: which account this is for, which section is this for, and which integration ID is this for? Because this is the holy trinity of being able to pinpoint one unique installation.

For this customer, Maltex Plastics predated migration to UUID-based account IDs, so the account ID was literally the customer name: `Maltex Plastics`.

### Postman Configuration

The collection requires setting up authentication with the Delete Integration Secret Key in the authorization header. Environment variables were not fully pre-configured, so manual values needed to be entered:

- **Account ID**: `Maltex Plastics` (legacy name)
- **Section ID**: `16134`
- **Integration ID**: `Lime`

### Initial Failure: Wrong Account ID

[Michal Rosikiewicz] initially attempted the deletion with an incorrect account ID. The request returned a 404: `Integration was not found`.

[Erik Andersson]: It's interesting that you got the 401 error though. Let me check what their actual credentials are.

After reviewing the Integration Manager, Erik identified the correct account name was `Maltex Plastics` (with correct spelling and case), not what had been initially provided.

[Erik Andersson]: Yeah, they have just given us the incorrect account ID.

### Squid Proxy Edge Case

During the actual deletion, the team encountered a known edge case:

[Erik Andersson]: Now we might run into the annoying edge case — this squid proxy that we have. Occasionally, very rarely, when you try to uninstall, the database refuses to delete the squid proxy entry. It just hangs on that request for whatever reason.

Fortunately, this instance succeeded, but the squid proxy issue is a documented gotcha that can cause hangs during deletion.

### Successful Deletion with External System Error

When the deletion executed, it encountered an HTML gateway error from the Lime CRM system (indicating misconfiguration on their end):

[Erik Andersson]: We get like HTML and a gateway error. So like they have some weird configuration here. But now we have completely disregarded this. We have deleted everything inside of Apsis.

The deletion succeeded because the `ignore_external_system_errors` flag was set. The system:
- Deleted all Apsis-side database entries
- Removed stored credentials
- Made the integration available for reinstallation

However, it did **not** delete webhooks in the Lime CRM, since those systems were unreachable.

---

## Post-Deletion Communication Plan

After confirming successful deletion, the team planned customer communication:

[Erik Andersson]: However, this also means that we have not deleted the web hooks in their CRM. So now everything is done in Apsis. Now we just need to ask SOC to ask the customer to do the webhook deletion.

The communication would inform the customer that:
1. Apsis-side deletion is complete
2. They must manually remove webhook configurations from their Lime CRM
3. If they don't, the CRM will continue sending data to Apsis endpoints (though it will be rejected)

---

## Why Force Deletion Exists: Multiple Scenarios

[Erik Andersson] outlined the reasons why the `ignore_external_system_errors` flag is necessary:

> This is valuable because we manually delete things if: there are no users in the account (so we can't delete event listeners from Audience), the account has been terminated (so we can't exchange delegation keys), or the CRM has some misconfiguration. That's why we have the `ignore_external_system_errors` flag.

---

## Issue Tracking and Customer Support Process

### Zero-Point Stories for Customer Issues
[Michal Rosikiewicz] explained the team's approach to tracking ad-hoc customer requests:

> If we have something from a customer that we have to deal with, we usually put it on the board and estimate it to zero points. This is because then we can take it the next day or day after, and we know that we have to deal with it and complete the story to satisfy the customer. Otherwise we can have multiple issues that nobody will even know we have. We cannot track it.

[Erik Andersson] noted that in smaller teams (3-4 people) with a rotating support schedule, this might not be necessary, but for larger teams it provides valuable tracking and historical context for future issues.

### Knowledge Base from Historical Issues
[Michal Rosikiewicz]: Some time ago, because we had Shortcut, we could search for similar issues from customers and build knowledge this way.

---

## Outstanding Issues and Next Steps

### 1. Dynamics 365 Domain/CORS Policy Question
An unidentified Dynamics customer has asked about domains that Apsis can be reached on, possibly for CORS policy configuration. This requires clarification:

[Erik Andersson]: I have no idea what's that. I don't know if this has some endpoint inside the CRM or the plugin or what's going on.

This will be investigated in a follow-up meeting.

### 2. CRM Implementation Consultant Sidetracking Support
[Erik Andersson]: Denis or someone like one of these people that do CRM implementation are very fond of trying to sidetrack SOC and support. He has reached out and told me there are some issues, but I'm trying to get him to go through the proper flow.

Verification needed on whether this is an integration issue or an Audience issue, with likely investigation required during the week.

---

## Key Takeaways

1. **Clear Integration Endpoint** is a powerful tool for forced deletion when external systems become unreachable or misconfigured, using the `Delete Integration Secret Key` from Secrets Manager.

2. **The "Holy Trinity"** of Account ID, Section ID, and Integration ID must be correctly specified to pinpoint the unique installation being deleted.

3. **Webhook cleanup is customer responsibility** after forced deletion — Apsis cannot reach the external CRM to remove them. Clear communication is essential to prevent accidental data leakage.

4. **Edge cases like squid proxy hangs** are known but rare. The system handles them gracefully with the ignore errors flag.

5. **API Key rotation doesn't require uninstallation** — customers can update credentials through the UI and continue using the integration.

6. **Proposed UI improvement**: A "Force Delete" button with appropriate disclaimers could empower customers while maintaining data privacy guardrails.

7. **Zero-point story tracking** for ad-hoc customer issues provides valuable historical context and ensures nothing falls through the cracks, even in smaller teams.

8. **Postman collections** are essential debugging tools — each integration type (FCC, Generic, Lime, Dynamics) has collection endpoints for testing and verification.

---

## Unresolved Questions and Action Items

- [ ] Clarify the Dynamics 365 customer's question about Apsis domains and CORS policy configuration
- [ ] Verify whether Denis's reported issues are integration or Audience related
- [ ] Future optimization: Consider replacing `Delete Integration Secret Key` with a proper delegation key
- [ ] Design and implement "Force Delete" button in UI with appropriate disclaimers and webhook cleanup warnings
- [ ] Ensure customer (Maltex Plastics) removes webhook configurations from Lime CRM
- [ ] Document the squid proxy edge case behavior and mitigation strategy