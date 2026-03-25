---
source_file: Erik - Integration statistics for Back Office kick-off.txt
domain: Apsis One Integrations
topics: [Webhook Configuration Issues, Full Sync Error Handling, Producer-Consumer Architecture, CRM System Integration Debugging, Error Propagation]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [FSC Enterprise, Webhook Registration, Full Sync Job, Producer Task, Consumer Task, Sync Status Management, Error Handling in Defer Functions]
session_type: knowledge-transfer
subdomains: [Architecture, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session documents two critical integration issues encountered during FSC Enterprise installations and full sync operations in on-premises environments. The first case covers a webhook registration conflict that was resolved by identifying and removing pre-existing webhooks in the customer's system. The second case reveals a systemic error handling bug in the producer-consumer architecture where sync jobs fail to properly signal completion, causing the consumer to wait indefinitely (up to 8 hours) for data that will never arrive. Both issues required investigation and have led to actionable development tickets for improving error propagation and sync status management.

## Case 1: FSC Enterprise Webhook Registration Conflict

### Initial Problem Report

A professional services consultant in the Benelux region reported issues installing FSC Enterprise in an on-premises environment. During the webhook registration step, the system returned an error indicating a page conflict—a type of error Erik had never encountered before.

> The issue was that during installation steps when we were trying to register webhooks, we received this error that the page was not displayed because there was a conflict.

[Erik Andersson]: The error appeared to originate from the CRM system rather than Apsis One itself, though this wasn't immediately clear.

### Debugging Approach: Three-Step Verification

[Erik Andersson]: When encountering environment issues, three basic diagnostic steps can be taken regardless of the CRM system:

1. **Network Accessibility**: Verify that the environment can be reached at all. This is particularly important for on-premises installations with IP whitelisting, which can be validated through manual requests without CRM team involvement. Check for timeout errors or permission denied responses.

2. **Permission Verification**: Confirm correct permissions on the API key. Specifically check whether the integration has permission to read contacts, read schema, and perform other required operations. Errors here typically manifest as 403 (permission denied) responses rather than proper data.

3. **Data Validation**: Verify that the environment returns proper data rather than HTML error pages. This confirms the integration is communicating correctly with the CRM system.

### Root Cause: Pre-existing Webhooks

[Erik Andersson] had direct access to someone who could access the customer's environment and database, which enabled collaborative debugging. When testing the installation flow, they discovered that webhooks for consent and contacts were already configured in the environment.

> When we deleted those two webhooks for the consent and contacts that existed in their environment, then the installation process could proceed.

[Erik Andersson]: This revealed that FSC Enterprise has a limitation preventing webhook installation if webhooks already exist in the environment. The origin of these existing webhooks remains unclear—possibilities include:
- A previous failed installation attempt that registered webhooks but failed before completion
- Manual webhook configuration or migration from an older version
- Migration from FSC Enterprise 12.0 to 12.1 that left legacy webhooks

Once the pre-existing webhooks were deleted, the installation completed successfully and Erik was able to retrieve metadata and contacts.

### Webhook Deletion Capability and Limitations

[Erik Andersson]: Apsis One does have the capability to delete webhooks from an environment, but this presents a problem: **if the installation fails during webhook registration, the system doesn't know the webhook IDs that need to be deleted**. In this case, the ID information was available only because Eric (the customer contact) could log into the system and provide them directly.

### Workaround and Documentation

For FSC Enterprise webhook registration conflicts:
- **Check for existing webhooks** in the environment
- **Delete pre-existing webhooks** before attempting installation
- **Verify proper completion** by checking whether webhook installation succeeds

[Erik Andersson]: This debugging methodology applies across CRM systems—Dynamics, Lime, Efficy Enterprise 12.0, and Tribe—making it a general principle for initial integration troubleshooting.

## Case 2: Full Sync Job Timeout Due to Improper Error Propagation

### Initial Symptom

A customer reported that a sync terminated after several hours, likely due to timeout. [Erik Andersson] recalled a similar issue previously identified where **consent exports that failed repeatedly would cause the sync to timeout after a couple of hours because the producer terminated itself without the sync being marked as failed**.

### Architecture Context: Producer-Consumer Model

[Erik Andersson]: The full sync job operates with a producer-consumer architecture:
- The **producer** retrieves data from the CRM system and places it into a queue
- The **consumer** pulls messages from the queue and processes them
- These are independent tasks with different termination conditions

The producer-consumer design includes a metadata field in the sync status blob:

```
status: [numeric value]
total_is_known: [0 or 1]
```

The `total_is_known` field signals to the consumer: *"The producer has finished. No more data will arrive in the queue. If the queue is empty and total_is_known is true, the sync is complete."*

### The Core Bug: Broken Error Propagation

In both observed cases, when the producer encountered a critical error:

1. The producer received an error from FSC Enterprise (e.g., when downloading contacts, returning a large HTML error blob instead of valid data)
2. The producer shut itself down
3. **However, the sync was never marked as failed**, and `total_is_known` was never set to true
4. The consumer, unaware that the producer had crashed, continued waiting for data

[Erik Andersson]: The consumer has termination logic that waits for an "obscenely long time" (approximately 8 hours) before giving up. It checks:
- Has `total_is_known` been set? (Producer finished)
- Is there data in the queue? (Continue processing)
- Has the job been cancelled? (Stop waiting)

If none of these conditions are met, the consumer waits indefinitely until the timeout threshold.

### Evidence from Logs

[Erik Andersson] examined the sync logs and found that:
- The sync was always marked as **succeeded** in the logs, never as failed
- This suggested the error encountered by the producer was not being properly passed to the cleanup/status-update function
- The pattern appeared in the "report" section at the end of the sync logs, showing the sync considered itself completed when it actually failed

### Technical Root Cause: Defer Function Not Receiving Errors

[Erik Andersson]: In Go, the `defer` function executes after a function completes. The producer's main function (called `everything()`) uses a defer to handle cleanup and status reporting:

```
func everything() {
  // ... main processing logic ...
  defer {
    // This runs at end, should set status/total_is_known
    // But errors aren't being properly passed here
  }
}
```

The problem: **Errors occurring in the producer are not being properly passed to this defer function**, so the status and `total_is_known` fields never get updated.

### Two Manifestations, Same Root Cause

**Case 1 (Consent Export):** The consent export failed after reaching maximum retries, producing an error that terminated the sync without marking it failed.

**Case 2 (Contact Download):** An error occurred during contact download from FSC Enterprise, terminating the producer without proper status update.

Both cases result in the same broken state: producer dead, `total_is_known` not set, consumer waiting until timeout.

## Proposed Solution: Error Handling in Cleanup

[Erik Andersson]: The fix requires two changes:

1. **In the defer function**: Check if an error occurred. If so, set the sync status to **failed** and set `total_is_known` to true.

2. **In the consumer termination logic**: Expand the sanity check to recognize not just `cancelled` status but also `failed` status. When either is set, immediately terminate the consumer instead of waiting.

```
Consumer checks:
- Is status cancelled? → Stop
- Is status failed? → Stop  [NEW]
- Is total_is_known true AND queue empty? → Stop
- Otherwise, wait (up to 8 hours)
```

[Michal Rosikiewicz] suggested potentially introducing a new status value like `producer_shutdown_improperly`, but [Erik Andersson] noted that the existing `failed` status is appropriate for this case. Failed is typically used for consumer errors, but should also be used when the producer fails.

### Benefits of This Fix

- Sync jobs that fail in the producer will now fail immediately rather than hanging for 8 hours
- Significant reduction in wasted ECS task execution time (currently 2 ECS tasks running for 8 hours unnecessarily)
- Improved customer experience—failures are reported rather than silently timing out
- Does not address the underlying CRM errors (e.g., why FSC Enterprise returns HTML instead of data), but properly signals them to the user

### Relationship to Root CRM Errors

[Erik Andersson]: While this fix prevents the hanging issue, it does not solve why FSC Enterprise returns errors in the first place. That requires investigation by the CRM development team. However:
- If FSC Enterprise returns an error, the sync will now fail immediately with a clear error message
- Users will know their sync failed rather than assuming it's still processing
- The infrastructure won't waste resources on doomed syncs

## Known Issues and Limitations

### Dependency on CRM Team for Root Causes

[Erik Andersson]: Currently, 5 unanswered issues exist for FSC Enterprise installations where problems originate in the CRM system. In an ideal workflow, support or a ticketing system would:
1. Detect an external system error at a specific code location
2. Immediately forward the ticket to the CRM developer team
3. Include relevant context about what the CRM system returned

### Webhook Deletion Workflow Gap

When a webhook conflict prevents installation, manual intervention is required to identify webhook IDs if the installation fails. The system should ideally provide a way to query and delete webhooks from the environment without direct database access.

### Pre-existing Webhooks After Version Migration

The cause of pre-existing webhooks in the failing customer's system remains unresolved. This could indicate:
- A gap in migration procedures from FSC Enterprise 12.0 to 12.1
- Incomplete cleanup from failed installation attempts
- Unclear documentation about webhook management during upgrades

## Development Status

[Tomasz Kowalski] confirmed that work on the sync error handling has already begun. [Erik Andersson] indicated that:
- A story for the producer-consumer error propagation issue already exists on the board
- The story should be ready for development
- It can be refined with more technical details about what needs to be set in the defer function
- Timeline: Post-Christmas sprint recommended due to team availability

## Key Takeaways

1. **Webhook Registration Debugging**: On-premises FSC Enterprise installations should be checked for pre-existing webhooks before attempting new webhook registration. The conflict error suggests webhooks already exist in the environment.

2. **Error Propagation is Critical**: In producer-consumer architectures, errors in the producer must be explicitly propagated to status management. Without this, failures manifest as timeouts rather than immediate failure signals.

3. **Three-Step Network Debugging Applies Broadly**: The verification approach (reachability → permissions → data validity) works across all CRM systems and doesn't require CRM team involvement for initial triage.

4. **CRM Errors Need Clear Escalation**: When Apsis One receives errors from external CRM systems (especially non-standard responses like HTML error pages instead of data), these should be clearly marked and escalated to the CRM development team with minimal filtering.

5. **Infrastructure Cost of Hanging Syncs**: Currently, failed syncs hang for up to 8 hours, consuming ECS task resources unnecessarily. Fixing error propagation will reduce infrastructure costs while improving user experience.

6. **Version Migration Gotcha**: FSC Enterprise 12.0 to 12.1 migrations may leave webhook artifacts that prevent new installations. Investigation needed into whether this is a known artifact or configuration-specific.

---

## Unresolved Questions & Action Items

| Item | Status | Owner | Priority |
|------|--------|-------|----------|
| Why did pre-existing webhooks exist in the customer's environment? Were they from a failed 12.0→12.1 migration? | Unresolved | CRM Team investigation needed | Medium |
| What causes FSC Enterprise to return HTML error pages instead of valid data in contact export? | Unresolved | CRM Team (5 similar open issues) | High |
| How should webhook IDs be discovered and deleted when installation fails with conflict error? | Unresolved | Feature request | Medium |
| Implement error propagation fix in producer defer function to set `failed` status and `total_is_known` | In progress | Development team | High |
| Expand consumer termination logic to check `failed` status in addition to `cancelled` | Pending | Development team | High |
| Refine existing story on board with specific defer function implementation details | Pending | Erik Andersson | Medium |