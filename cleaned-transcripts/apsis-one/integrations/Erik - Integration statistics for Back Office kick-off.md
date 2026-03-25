---
source_file: Erik - Integration statistics for Back Office kick-off.txt
domain: Apsis One Integrations
topics: [Enterprise Webhook Installation Issues, Sync Timeout Problems, Producer-Consumer Architecture, Error Handling in Sync Jobs, Full Sync Process Debugging]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [FSC Enterprise, Webhook Registration, Contact Sync, Consent Export, Producer Task, Consumer Task, Sync Status Management, Error Handling in Golang defer]
session_type: knowledge-transfer
subdomains: [Architecture, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Outbound Flow]
---

## Session Overview

This knowledge-transfer session covers two critical issues encountered during Apsis One integration work with FSC Enterprise: (1) a webhook installation conflict on an on-premises environment that was resolved by identifying and deleting pre-existing webhooks, and (2) a recurring sync timeout problem where the producer crashes on error but fails to properly signal completion to the consumer, causing the consumer task to hang for hours. Both issues are rooted in how the sync job's producer-consumer architecture handles error states and status propagation.

---

## Case 1: FSC Enterprise Webhook Installation Conflict

### Issue Description and Diagnosis

A professional services consultant in the Benelux region reported a failure while installing FSC Enterprise (version not initially specified, but context suggests 12.0 or 12.1) in an **on-premises environment**. During webhook registration as part of the installation process, the system returned a "page not displayed" error indicating a conflict.

[Erik Andersson]: The issue appeared to be an internal Apsis problem at first, but investigation revealed it was actually a limitation or conflict within the FSC Enterprise system itself, which typically requires escalation to CRM developers.

### Three-Step Debugging Methodology

[Erik Andersson] outlined a standardized three-step approach for diagnosing integration issues when direct CRM developer support is unavailable:

1. **Network/Access Verification**: Confirm the integration can reach the CRM environment at all. On-premises deployments frequently implement **IP whitelisting**, so the first step is making a manual request to verify:
   - No timeout errors occur
   - No "permission denied" responses are returned

2. **Permission Verification**: If the environment is reachable, verify correct API key permissions:
   - Can read contacts?
   - Can read schema?
   - Environment returns proper data or HTML error pages?

3. **Data Validation**: Confirm the environment returns properly formatted data rather than error HTML.

### Root Cause: Pre-Existing Webhooks

Through direct access to the customer's environment and database (unusual circumstance allowing preliminary debugging), Erik and the customer's contact discovered the real issue: **the FSC Enterprise environment already had webhooks configured** for consent and contacts.

> The error is not something that is standard in the general connector—we typically don't accept these errors that they responded to, but when we deleted those two webhooks for the consent and contacts that existed in their environment, then the installation process could proceed.

[Erik Andersson]: FSC Enterprise appears to have added a limitation that prevents installing webhooks if webhooks already exist in the environment for those entities.

### Unknown Migration Path

The root cause of the pre-existing webhooks remains unclear. Possible explanations:
- A previous installation attempt that registered webhooks but failed to complete
- Manual migration from FSC Enterprise 12.0 to 12.1 where old webhooks (named `FSC Enterprise`) were renamed (e.g., to `FSC Enterprise-2`) instead of deleted
- Webhooks from an older environment configuration

### Resolution Steps

Once the two existing webhooks were deleted manually, the installation proceeded successfully. Erik was able to:
- Retrieve metadata
- Receive contact data
- Complete full verification of the integration

### Operational Gap: Webhook Cleanup Without IDs

[Erik Andersson] identified a limitation in the standard process: if webhook installation fails and the conflict error is returned, **the system doesn't know which webhook IDs need to be deleted**. In this case, manual environment access allowed the customer to provide the IDs. 

> We do have the possibility to delete webhooks from the environment. The issue is that if the installation fails or like if we get this error during the webhook installation, we don't know the IDs of the webhooks to delete.

**Applicability**: This debugging methodology (network access → permissions → data validation) applies across all CRM systems: Dynamics, Lime, FSC Enterprise 12.0, Tribe, etc.

---

## Case 2: Sync Timeout Due to Improper Error Handling in Producer-Consumer Architecture

### Issue Summary

A customer reported that a full contact sync terminated after several hours, likely due to a timeout. The root cause was not a timeout per se, but rather an **improper error state** in the sync job's producer task that was never communicated to the consumer task, causing the consumer to wait indefinitely.

### Producer-Consumer Architecture Overview

[Erik Andersson] explained the architecture:
- The sync job consists of **two independent tasks**: a **producer** and a **consumer**
- The **producer** downloads data from the CRM system (e.g., contacts)
- The **consumer** processes messages from a queue populated by the producer
- The consumer relies on two key pieces of information to know when to stop:
  1. **`total_is_known`** field: Set by producer to indicate "I have downloaded everything I will download"
  2. **Queue emptiness**: Consumer checks if the queue is empty AND `total_is_known` is true

### The Failure Scenario

In this case:
1. The producer received an **error from FSC Enterprise** when attempting to download contacts (a critical error showing an HTML blob instead of data)
2. The producer crashed and shut down itself
3. **Critical failure**: The producer never set `total_is_known = true` before shutting down
4. The producer also never set the sync status to `failed`
5. The consumer, unaware of the producer's failure, continuously checked the queue and waited for more messages
6. Since `total_is_known` remained false, the consumer assumed the producer was still working (timeout or temporary issue) and kept waiting

### Consumer's Timeout Clause

The consumer has a built-in safeguard: if it receives no messages for approximately **8 hours**, it terminates itself, even if `total_is_known` is false. However, this means customers experience an unnecessary 8-hour delay before seeing the sync marked as failed.

[Erik Andersson]: The consumer has termination clauses including checking if `total_is_known` is known, but the critical issue is that if the producer shuts down improperly, it never updates this field.

### Code-Level Root Cause

[Erik Andersson] identified the issue in the **`everything` function** in the producer, which contains all sync logic:

```
everything() {
  // ... main logic
  defer {
    // Cleanup function executed after everything else
    // PROBLEM: Error state is not being properly passed here
    // total_is_known is never set to true on error
    // sync status is never set to failed
  }
}
```

**The Problem**: The `defer` function in Golang (executed after the enclosing function completes) is not checking whether an error occurred. If an error happens during producer execution:
- The producer shuts itself down (no more messages written to queue)
- The `defer` cleanup runs but doesn't have access to the error that caused the shutdown
- `total_is_known` remains false
- Sync status is never marked as failed
- Consumer waits indefinitely (up to 8 hours)

[Erik Andersson]: We are never seeing any full sync failures in the logs—everything is always marked as succeeded. I don't think the error that occurred is being properly passed on to the cleanup function.

### Similar Historical Issue: Consent Export Retry Exhaustion

[Michal Rosikiewicz] and [Erik Andersson] noted this is a **variation of a previously identified problem**. In that case:
- Consent export failed repetitively
- After reaching max retries, an error terminated the sync (the producer)
- Same result: `total_is_known` was never set, consumer hung
- Different code location, same architectural issue

### Proposed Solution

The fix must occur in the `defer` cleanup function:

1. **Check for error condition**: Determine if the producer encountered an error
2. **Set status to `failed`**: Update the sync job status (not succeeded, but failed)
3. **Set `total_is_known = true`**: Signal to the consumer that no more data is coming
4. **Propagate the actual error**: Include the error details in the sync status

```
if error_occurred:
  - set status = "failed"
  - set total_is_known = true
  - include error details
```

This allows the consumer to:
- Check not just `total_is_known`, but also sync status
- Exit immediately if status is `failed` or `cancelled` (not just wait for timeout)
- Avoid unnecessary resource consumption (two ECS tasks running for 8 hours when sync failed immediately)

[Erik Andersson]: Failed is typically something we do if something goes wrong in the consumer, but we are not setting it if something goes wrong in the producer, which of course it should also be done.

### Existing Story

A Jira story already exists to address this issue. [Erik Andersson] committed to making the story more detailed with specific guidance on what needs to be set in the `defer` function.

### Business Impact

- **Customer Experience**: Syncs that fail immediately appear to hang for up to 8 hours
- **Resource Waste**: Two ECS tasks run unnecessarily for 8 hours per failed sync (not "super big" tasks but still wasteful at scale)
- **Observability**: Failures are masked in logs (always show "succeeded" even when they fail)
- **Cascading Issues**: This is a recurring pattern affecting multiple error scenarios

---

## Distinction Between CRM System Errors and Apsis Code Defects

[Erik Andersson] emphasized a critical operational distinction:

**CRM System Errors**: When FSC Enterprise (or other CRM systems) returns errors or behaves unexpectedly, these must be escalated to CRM developers. Apsis cannot fix how the external system behaves.

**Apsis Architectural Issues**: When Apsis code fails to handle errors from external systems gracefully, that is an Apsis responsibility. The sync timeout issue is **not** about why FSC Enterprise returned an error; it's about **how Apsis reacts to that error**.

> If it had been like anything inside of Apsis, I would have taken care of that immediately. But the issue which you will unfortunately see a lot of right now is that the problems are essentially every time in the CRM system.

Current process gap: When integration issues occur, they often originate in the CRM system, and Apsis support lacks a formal escalation process to CRM developers. [Erik Andersson] mentioned preparing communication to CRM development leads (Ali and Lukasz) to establish a proper escalation procedure, noting 5 open FSC Enterprise issues currently without developer response.

---

## Debugging Best Practices Highlighted

### When Direct Access Is Available
In the webhook case, having direct access to the customer's environment and database enabled rapid root-cause analysis that would otherwise have required CRM developer involvement. However, this is rare and should not be the standard path.

### Error Message Interpretation
Large HTML error blobs (like those returned by FSC Enterprise) indicate the CRM system encountered an error, not that the integration request was malformed. This is distinct from a properly formatted error response.

### Log-Based Diagnosis
If sync jobs are always marked "succeeded" in logs but customers report timeouts, this is a strong indicator that errors are being silently swallowed and not properly reflected in sync status fields.

---

## Key Takeaways

1. **FSC Enterprise Webhook Limitation**: FSC Enterprise 12.0/12.1 does not allow webhook installation if webhooks already exist for those entities. If this error occurs, check for and delete pre-existing webhooks (consent, contacts) before retrying installation.

2. **Webhook Conflict Troubleshooting Methodology**: Apply the three-step debugging approach (network access → permissions → data validation) before escalating to CRM developers. This works across all CRM systems.

3. **Producer-Consumer Sync Architecture**: The sync job relies on the producer correctly setting `total_is_known` and sync status when complete or failed. If the producer crashes without updating these, the consumer will hang.

4. **Error Propagation in Golang defer**: The `defer` cleanup function in the producer's `everything()` function must explicitly check if an error occurred and update sync status accordingly. Currently, errors during producer execution are not being passed to the cleanup function.

5. **Sync Status Missing Failed State**: Sync jobs are only marked failed in consumer errors; producer errors are not reflected in status. This must be fixed to prevent 8-hour timeouts.

6. **Recurring Pattern**: The contact sync timeout and the previously identified consent export timeout share the same root cause: improper error handling in the producer's cleanup phase. The fix applies to both.

7. **CRM Error Escalation Gap**: A formal escalation process to CRM developers is needed for issues originating in external systems (FSC Enterprise, etc.). Currently lacking.

8. **Resource Impact**: Unaddressed sync timeouts cause unnecessary ECS task execution (up to 8 hours per failure), impacting cost and observability at scale.

---

## Unresolved Questions & Action Items

**Questions Requiring CRM Developer Input**:
- Why did the webhook conflict error occur? Is this a known limitation of FSC Enterprise installation?
- Why did FSC Enterprise return an HTML error blob instead of proper contact data in the second case?
- How did the pre-existing webhooks get into the customer's environment? (Possible migration issue?)

**Action Items**:
1. **[Erik Andersson]**: Create/refine Jira story for producer error handling fix with specific details on `defer` function updates. Mark as Q1 or soon after Christmas break.
2. **[Erik Andersson]**: Update consumer status checking logic to also check for `failed` status (currently only checks `cancelled`).
3. **[Erik Andersson]**: Draft formal escalation procedure email to Ali and Lukasz (CRM development leads) to establish process for external system errors.
4. **Team**: Review both cases and ensure webhook conflict troubleshooting is documented in support runbook.

---

## Technical Debt Noted

- Error handling in producer's `defer` function is incomplete
- No clear path from Apsis support to CRM developer escalation
- Consumer's termination timeout is overly long (8 hours) for unresponsive producer states
- Sync status field doesn't differentiate between producer and consumer failures