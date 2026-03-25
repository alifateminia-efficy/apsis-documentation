---
source_file: Erik - Integration statistics for Back Office kick-off.txt
domain: Apsis One Integrations
topics: [Webhook Registration Errors, Fiber Connector Enterprise Installation, Sync Timeout Issues, Producer-Consumer Architecture, Error Handling in Full Sync Jobs, On-Premises CRM Environments]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Fiber Connector Enterprise 12.0/12.1, Webhook Registration System, Full Sync Job Architecture, Producer/Consumer Tasks, Contact Download Process, Consent Export Process, ECS Tasks, Sync Status Management]
session_type: knowledge-transfer
subdomains: ['Architecture', 'Efficy Enterprise 12.0 Integration', 'Efficy Enterprise 12.1 Integration']
---

## Session Overview

This knowledge transfer session documents two critical issues encountered during Apsis One integration work: (1) a webhook registration conflict during Fiber Connector Enterprise installation in an on-premises environment in the Benelux region, and (2) a sync timeout problem caused by improper error handling in the producer-consumer architecture of the full sync job. The session explains the debugging methodology for CRM integration issues, the root causes of both problems, and the required fix for the timeout issue.

---

## Webhook Registration Conflict During FCE Installation

### Initial Problem Report

[Erik Andersson]: The issue was reported by a professional services consultant in Benelux (Netherlands/Central Europe) attempting to install Fiber Connector Enterprise (FCE) 12.1 in an **on-premises environment**. During the webhook registration step, the system returned a "page not displayed" error due to a conflict. This was initially unclear as to the root cause.

> "The error was easier than you could think."

[Michal Rosikiewicz] confirmed this was an "entire system conflict."

### Investigation Methodology for CRM Integration Issues

When troubleshooting CRM system integration problems, there are three basic diagnostic steps that can be performed without direct CRM developer involvement:

1. **Network Reachability Check**: Verify if the environment can be reached at all, particularly important for on-premises deployments with IP whitelisting.
   - Test with manual requests to check for timeouts or "permission denied" responses
   - On-premises environments commonly have restrictive network policies

2. **Permission Verification**: Confirm the integration has correct permissions on the CRM key.
   - Verify ability to read contacts
   - Verify ability to read schema
   - Check that the environment returns proper data rather than HTML error pages

3. **Data Validation**: Ensure the environment returns valid structured data rather than error responses.

[Erik Andersson]: These diagnostic steps apply across all CRM systems: Dynamics, Lime, Efficy Enterprise 12.0, Tribe, and others.

### Root Cause: Existing Webhooks Conflict

[Erik Andersson] was given direct access to the customer's environment and configuration through an on-site contact. When attempting the installation flow, the team discovered the environment **already had webhooks configured** (specifically webhooks for "consent" and "contacts").

**Key finding**: Fiber Connector Enterprise 12.1 has a limitation that **prevents installing new webhooks if webhooks already exist in the environment**. This limitation was not previously known.

The pre-existing webhooks were deleted, after which the installation proceeded successfully. The integration could then:
- Retrieve metadata
- Download contacts
- Function correctly

### Webhook Origin Unknown

[Michal Rosikiewicz]: The source of the pre-existing webhooks was unclear. Possible explanations:
- Previous failed installation attempt that registered webhooks but failed before completing
- Attempted manual installation that succeeded only in creating the webhooks

[Erik Andersson]: Migration from FCE 12.0 to 12.1 was considered as a possibility (old webhooks would be named "FC Enterprise" vs "FC Enterprise Dash 2"), but this seemed unlikely.

### Problem Resolution and Future Prevention

[Erik Andersson]: 
> "If we receive this error from enterprise during the installation step for the web hooks then the important thing to check is: are there existing webhooks for the environment and if so they need to be deleted."

**Caveat**: The Apsis One integration has the ability to delete webhooks from the environment, but **if installation fails during webhook registration, we don't know the webhook IDs to delete**. In this case, direct access to the CRM system allowed the customer to provide the webhook IDs manually.

This remains a limitation when direct customer access is unavailable—the issue would need to be escalated to CRM developers for webhook removal.

### CRM System Issues vs. Integration Issues

[Erik Andersson] emphasized a critical pattern: Most current integration failures originate in the CRM system itself, not Apsis One. When an error comes from the CRM system, it should be immediately escalated to the CRM developer team rather than handled by the integration team.

At the time of this session, there were **5 unanswered issues for FCE Enterprise** blocking proper integration function. The ideal workflow is:
1. Support/integration team identifies external system error
2. Issue is forwarded to CRM developer team immediately
3. CRM team provides resolution

---

## Sync Timeout Issue: Producer-Consumer Architecture Failure

### Symptom Description

A customer reported that a sync task **terminated after several hours**, most likely due to a timeout. This was a recurring issue that had been encountered previously with consent exports.

### Root Cause: Error Handling Doesn't Properly Set Sync Status

The Apsis One full sync job uses a **producer-consumer architecture**:

- **Producer task**: Retrieves data from the CRM system (contacts, consent records, etc.)
- **Consumer task**: Processes messages from the queue and writes to Apsis One

When the producer encounters a critical error from the CRM system and shuts down, **the sync status is not properly marked as failed**. This causes the consumer to remain in a waiting state.

### Technical Details: Consumer Termination Logic

The consumer has multiple termination clauses:

1. **Timeout clause**: If no messages arrive for approximately 2 hours, the consumer terminates itself (to prevent indefinite waiting)
2. **Completion clause**: If `total_is_known` is set to true AND the queue is empty, the consumer terminates (indicating the producer has finished and sent all data)

**The problem**: When the producer fails:
- No messages are added to the queue
- The `total_is_known` field is NOT set (remains false)
- The producer shuts down without updating sync status

Result: The consumer doesn't recognize that the producer has stopped. It waits indefinitely for more messages, checking repeatedly: "Is `total_is_known` set? No. Is the queue empty? Yes. Therefore, wait longer for more messages."

The consumer only terminates after hitting the upper timeout limit (~8 hours), wasting ECS task execution time.

### Two Similar But Distinct Failure Modes

**Case 1 (Previously encountered)**: Consent export failed repetitively, reached maximum retries, and terminated the producer.

**Case 2 (Current issue)**: Contact download from the CRM system returned an error (described as a "big HTML error BLOB" instead of proper data), causing the producer to crash.

Both cases result in the same outcome: improper producer termination without setting status fields.

### Code Location and Fix Strategy

The issue is in the **`everything` function** (the main function in the producer where all operations occur) and specifically in the **`defer` function** that executes after the main function completes.

[Erik Andersson]:
> "Somewhere in here we need to check: if an error occurred and what makes me believe that. The error is not being properly passed on. It's because we are never seeing any full sync failures in the logs. It is always marked as succeeded."

Currently, the `defer` function prints a report but does not check whether an error occurred. The fix requires:

1. **Check for error in the defer function**: Detect if the producer encountered an error during execution
2. **Set status to "Failed"**: Mark the sync job as failed (not succeeded)
3. **Set `total_is_known` to true**: Signal the consumer that the producer is done and no more messages will arrive

[Michal Rosikiewicz] suggested potentially introducing a new status value like "producer_shut_down_improperly," but [Erik Andersson] noted that the existing "failed" status is appropriate and should be set for producer errors (it's currently only set for consumer errors).

### Consumer Sanity Checks

The consumer currently checks:
- Has a message arrived recently?
- Is the queue empty?
- Is `total_is_known` set?
- Is the job status "cancelled"?

[Erik Andersson]: The consumer checks for "cancelled" status but should also check for "failed" status. If either is set, the consumer should terminate immediately instead of continuing to wait.

### Impact and Priority

- **Customer Impact**: Customers experience failed syncs that don't report as failures, taking up to 8 hours to timeout
- **Infrastructure Impact**: ECS tasks remain running for ~8 hours when they could complete in minutes
- **Priority**: Should be addressed relatively soon (estimated for Q1), as it causes both customer dissatisfaction and wasted execution resources

---

## Existing Issue Tracking

[Erik Andersson] noted that there is **already a story/ticket on the board** for this timeout issue. The same fix applies to both the consent export failure case and the contact download failure case, as they stem from identical error handling problems in the producer.

[Tomasz Kowalski] and [Erik Andersson] agreed the story should be marked as "ready for development."

---

## Key Takeaways

1. **Webhook Registration Gotcha**: Fiber Connector Enterprise 12.1 cannot install webhooks if any webhooks already exist in the environment. If this error occurs, check for and delete pre-existing webhooks before retrying installation.

2. **Debugging Methodology**: For any CRM integration issue, start with three basic checks (network reachability, permissions, data validity) before escalating to CRM developers.

3. **On-Premises Specific**: On-premises deployments commonly have IP whitelisting and other restrictions that complicate troubleshooting. Direct access to the environment is invaluable for diagnosis.

4. **Producer-Consumer Status Bug**: The full sync job's producer doesn't properly set the `total_is_known` and `status` fields when errors occur, causing the consumer to hang for up to 8 hours before timing out.

5. **Fix Required**: The producer's `defer` function must:
   - Check whether an error occurred during execution
   - Set sync status to "failed" if an error occurred
   - Set `total_is_known` to true to signal completion
   - The consumer must then check for "failed" status and terminate immediately

6. **CRM vs. Integration Boundary**: Most current issues originate in the CRM system. Establish clear escalation paths to CRM developers when external system errors are identified.

7. **Error Logging Gap**: Currently, all syncs are logged as "succeeded" even when they fail in the producer, masking the real issue. This logging must be corrected when the fix is implemented.

---

## Unresolved Questions & Follow-Up Items

- [ ] **Webhook Origin Investigation**: Determine how webhooks came to exist in the customer's environment (previous failed install vs. migration vs. manual creation). This could help prevent similar situations.
- [ ] **Producer Error Handling Fix**: Implement the defer function check and status/total_is_known setting (already has existing ticket on board)
- [ ] **Consumer Status Check Enhancement**: Update consumer termination logic to check for "failed" status in addition to "cancelled"
- [ ] **FCE 12.1 Webhook Limitation Documentation**: Document the webhook conflict behavior and add it to installation troubleshooting guide
- [ ] **Contact Download Error Investigation**: Once producer error handling is fixed, investigate why CRM returned HTML error blob during contact download (requires CRM developer involvement)

---

**Note**: This session took place on December 19, 2025, during the holiday period. Multiple team members were planning leave, with limited availability starting December 22 and resuming January 5, 2026.