---
source_file: Erik - Integration statistics for Back Office kick-off.txt
domain: Apsis One - Integrations
topics: [Webhook Installation Conflicts, On-Premise Environment Debugging, Full Sync Job Error Handling, Producer-Consumer Architecture, CRM System Integration Errors, Timeout Issues in Data Synchronization]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [FSC Enterprise, Apsis One, CRM Systems (Dynamics, Lime, FEC Enterprise), Webhook Management, Full Sync Job, Producer-Consumer Message Queue, Contact Export, Consent Export]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covers two critical integration issues encountered during back-office operations, with focus on debugging methodologies and architectural problems in the full sync job. The first case involves a webhook installation conflict in an on-premise FSC Enterprise environment that was successfully resolved through environmental access and manual configuration inspection. The second case identifies a systemic issue in error handling where producer task failures are not properly communicated to consumer tasks, causing syncs to hang for extended periods (up to 8 hours) before timing out. Both cases highlight the need for better error propagation in the defer cleanup function of the sync producer.

---

## Webhook Installation Conflict in On-Premise FSC Enterprise

### Initial Problem Report
A professional services consultant in the Benelux/Netherlands region reported issues during FSC Enterprise installation. The environment was on-premise, which typically introduces additional complexity due to network and security configurations. During the webhook registration step, the system returned a "page not displayed" error indicating a conflict.

[Erik Andersson]: The initial assessment was that this could be an internal Apsis issue, but investigation revealed the root cause was environmental rather than application-based.

### Debugging Methodology for External System Errors

When integration errors originate from external CRM systems, the standard procedure involves three basic verification steps that can be performed without CRM developer involvement:

1. **Network Connectivity Verification**: Confirm ability to reach the environment at all, particularly important for on-premise installations with IP whitelisting. This can be verified through manual requests checking for timeouts or permission denied responses.

2. **Permission Validation**: Verify correct permissions on the API key, such as read access to contacts or read access to schema. Proper data return vs. HTML error responses indicates whether permissions are correctly configured.

3. **Data Format Validation**: Confirm the environment returns proper structured data rather than error HTML.

[Erik Andersson]: These debugging steps can be applied consistently across any CRM system (Dynamics, Lime, FEC Enterprise, etc.) without requiring CRM developer involvement for initial triage.

### Root Cause Discovery

Through direct access to the customer's environment and database configuration, the issue was identified: **existing webhooks were already registered in the environment**. When attempting to install webhooks, FSC Enterprise had a limitation preventing webhook installation if webhooks already existed.

> The environment already had web hooks configured for consent and contacts that existed in their environment. This error suddenly made sense when thinking about it.

The pre-existing webhooks were likely from:
- A previous failed installation attempt where webhooks were registered but installation failed to complete
- A migration scenario (e.g., from FEC Enterprise version 12.0 to 12.1) where old webhooks remained

### Resolution

Deletion of the two existing consent and contacts webhooks allowed the installation to proceed. Post-deletion verification confirmed successful metadata retrieval and contact synchronization.

[Erik Andersson]: This represents an unusual case because we had direct access to the customer environment and configuration, which is not standard. Normally, the CRM developers would need to diagnose the error directly.

### Knowledge for Future Incidents

If the "page not displayed" conflict error is received during FSC Enterprise webhook installation, the first check should be whether webhooks already exist in the environment. These must be identified and deleted before retry.

**Limitation**: Without direct environment access, webhook IDs cannot be discovered if the installation fails during the webhook registration step. The CRM system needs to provide the webhook IDs or allow query access for existing webhooks.

---

## Producer-Consumer Sync Job Error Handling Defect

### Symptom: Extended Timeout Failures

A sync terminated after several hours (approximately 8 hours), most likely due to a timeout condition. The producer task shut itself down after encountering an error, but the consumer task remained active waiting for messages indefinitely.

### Root Cause Analysis

The full sync job architecture uses independent producer and consumer tasks:

- **Producer**: Retrieves data from the CRM system (contacts, consent exports, etc.) and writes messages to a queue
- **Consumer**: Reads messages from the queue and processes them
- **Communication**: The producer signals completion by setting `total_is_known = 1` in the sync job status, indicating "I have finished downloading all data"

[Erik Andersson]: When the producer encounters a critical error (e.g., CRM returns HTML error blob instead of expected data), the producer shuts itself down. However, the sync job status is never updated to reflect this failure.

### Consumer Termination Logic

The consumer has a termination clause that includes multiple conditions:

1. **Normal completion**: Queue is empty AND `total_is_known == 1`
2. **Timeout**: No messages received for approximately 8 hours
3. **Explicit cancellation**: Job status is marked as cancelled

[Erik Andersson]: The problem is that when the producer crashes, `total_is_known` is never set to true, and the status is never updated. The consumer interprets the lack of messages as a temporary delay (CRM timeout that will resolve) and continues waiting.

### Current Failure Mode

```
Producer encounters error → Producer shuts down → 
defer function prints report → No status update → 
Consumer sees: queue empty, total_is_known=false → 
Consumer logic: "Producer still working, might get more messages" → 
Consumer waits 8 hours → Timeout reached → Consumer terminates
```

**Observed pattern**: All sync jobs appear to succeed in logs even when producer failures occur, because the failure status is never propagated.

### Technical Root Cause

The issue exists in the `Everything` function (the main producer function in Golang where everything happens). The `defer` cleanup function that executes after the function completes is not properly checking for errors that occurred during execution.

> The error is not being properly passed on to the cleanup function. That's why we never see full sync failures in the logs—they're always marked as succeeded.

### Required Fix

In the defer cleanup function, add error checking:

1. **Detect producer errors**: Check if an error occurred during producer execution
2. **Set status to failed**: Mark the sync job status as `failed` (not just leave it unset)
3. **Set total_is_known**: Mark `total_is_known = 1` to signal the consumer that no more messages will arrive

[Erik Andersson]: The consumer already has sanity checks for `cancelled` status. We can expand this to also check for `failed` status, and if the job is failed, terminate immediately rather than waiting for the timeout.

### Related Historical Issue

A similar symptom occurred when consent exports failed repetitively, exhausted retry logic, and resulted in an error. The producer terminated without updating the job status, causing the same 8-hour timeout behavior in the consumer. The fix approach is identical despite different triggering errors.

### Impact and Priority

- **Performance impact**: ECS tasks run idle for 8 hours consuming resources, though not high-resource tasks
- **User experience**: Customers experience unexpected delays and unclear failure states
- **Suggested timeline**: Q1 (post-Christmas), as the issue occurs frequently but is not critical blocking

[Tomasz Kowalski]: A story already exists for this issue and should be ready for development with more detailed specification of what needs to be set in the defer function.

---

## CRM System Error Classification

### Errors Beyond Apsis Control

The current observation across enterprise issues is that **the vast majority of reported problems originate from the CRM system**, not from Apsis One code. These errors require CRM developer involvement to diagnose and resolve.

[Erik Andersson]: There are currently 5 open enterprise issues that are unanswered specifically because they require CRM system investigation. These cannot be resolved by the Apsis integration team.

### Ideal Workflow for External Errors

1. Integration code detects an error response from the CRM system
2. Error is immediately identified as external (CRM-sourced)
3. Support or integration team forwards the issue directly to CRM developers with:
   - The error context and full error response
   - Steps to reproduce in the customer environment
4. CRM team investigates and responds

**Current state**: This formal handoff process is not yet established, requiring ad-hoc coordination and escalation.

---

## Key Takeaways

1. **On-Premise Debugging Methodology**: Three-step verification (network, permissions, data format) can resolve most on-premise connectivity and access issues without CRM vendor involvement.

2. **Webhook Installation Gotcha**: FSC Enterprise prevents webhook installation if webhooks already exist. Check for and delete pre-existing webhooks before retry if installation fails.

3. **Producer-Consumer Decoupling Risk**: Failures in the producer task are not propagated to the consumer task, causing indefinite waits. This is a systemic issue affecting all sync jobs, not just specific error cases.

4. **Error Handling Fix**: The defer cleanup function in the producer must check for errors and set both `total_is_known` and `status` fields to enable consumer termination.

5. **CRM System Errors Are Common**: The majority of reported integration issues require CRM developer involvement and should be escalated directly rather than investigated by Apsis integration team.

6. **Debugging Access Matters**: Direct access to customer environments (database, configuration) dramatically accelerates root cause identification, but this is not the standard operational model.

---

## Unresolved Questions & Action Items

### Action Items

1. **Draft/Refine Story for Producer Error Handling**: Add detailed specification of what must be set in the defer cleanup function (status field, total_is_known field, specific values)
   - Assignee: Erik Andersson
   - Timeline: Post-Christmas, Q1

2. **Establish CRM Error Escalation Process**: Create formal handoff procedure for when errors are identified as CRM-sourced
   - Stakeholders: Erik Andersson, Ali, Lukasz, CRM development team
   - Current status: Email being drafted to formalize this

### Unresolved Technical Questions

1. **FSC Enterprise Webhook Limitation Origin**: Why does FSC Enterprise prevent webhook installation when webhooks already exist? Is this intentional design or a bug? Who made this decision?

2. **Pre-Existing Webhook Sources**: In the customer's case, how did webhooks initially get registered if the installation never completed? Need confirmation of whether this was a failed partial installation, migration artifact, or manual configuration.

3. **Consumer Status Checking**: Does the consumer currently check for `failed` status in its sanity checks, or only `cancelled`? Need to verify whether status check expansion is necessary or if only total_is_known is required.