---
source_file: Erik from Nov 27th 2025.txt
domain: Apsis One Integrations
topics: [Full Sync Architecture, Error Logging and Debugging, Exit Codes, Producer-Consumer Pattern, CRM Data Synchronization, Audience Consent Export, Race Conditions, CloudWatch Log Navigation]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, Redis, CloudWatch Logs, Audience Service, CRM System]
session_type: debugging-session
subdomains: [Architecture]
---

## Session Overview

This is a live debugging session focused on troubleshooting a failed full sync that resulted in exit code 2. Erik walks Michal through the architecture of the full sync system, explaining how to navigate CloudWatch logs to identify whether errors originate from the producer (CRM data extraction) or consumer (data delivery to Audience) components. The session reveals a race condition ("concurrent map writes") in the consumer, a timeout in the Audience consent export callback, and highlights architectural decisions around callback handling and consent optimization.

---

## Full Sync Architecture: Three-Layer Error Diagnosis

When a full sync fails, the source of the error can be identified by which stage has actually started:

### Full Sync Manager Layer

[Erik Andersson]: The **Full Sync Manager** is the synchronous entry point that initiates a full sync. If a full sync fails to start immediately (e.g., user clicks "full sync" button and it immediately fails), the error will be in the **Full Sync Manager log group**.

In the case being debugged, the sync had already begun counting profiles and producing messages, so the error was not at the manager level.

### Producer and Consumer ECS Tasks

[Erik Andersson]: Once a full sync starts, it spins up two separate **ECS tasks**:
- **Producer task**: Reads messages from the CRM system and puts them into a temporary queue
- **Consumer task**: Reads messages from the queue and sends them into Audience

The presence of profile counts and message totals in the logs indicates that production and consumption have actually started, meaning you must look in the Producer or Consumer log groups, not the Manager logs.

### Log Group Navigation Strategy

[Erik Andersson]: In staging environments, there are typically multiple log groups for the same full sync. To avoid checking every event from both syncs:
- Go to the **log group for full syncs, but not all events** (dedicated log streams)
- Each log stream represents one unique full sync, with each sync starting its own task
- The most recently started full sync will typically be the top log stream
- Identify the specific sync by its start timestamp

[Michal Rosikiewicz] and Erik confirmed that two syncs were running: one at 10:19 and another at 10:21.

---

## The Concurrent Map Writes Race Condition

### Error Details

During consumer initialization, the logs showed the following sequence:
- Environment setup and configuration loading
- Redis connection startup
- Worker thread initialization
- Consumer begins reading messages from queue

Then the fatal error occurred:

```
Concurrent map writes error
Fatal error encountered for profile 26216
```

[Erik Andersson]: This is strange because **this logic has not changed for 2 years**. The race condition suggests that two goroutines or threads are attempting to write to the same map concurrently without proper synchronization.

[Michal Rosikiewicz] questioned whether profile 26216 was recently modified or whether he had interacted with it. Erik clarified that during a full sync, **all profiles in the CRM are downloaded regardless of whether they were recently updated**, so the existence of the profile in the sync is not necessarily related to recent changes.

### Severity and Recurrence

[Erik Andersson]: This type of error does not seem to occur frequently, and the root cause is uncertain. The fact that the consumer logic unchanged for 2 years suggests this may be a very rare race condition rather than a logic error.

> "No idea what this would be... for sure it's not something that would occur frequently."

---

## Exit Code 2 and Task Stop Behavior

### Exit Code Definition Location

[Erik Andersson]: Exit codes are defined in the backend code. They are not visible in CloudWatch logs directly. The relevant code is in:

```
integration/
  full_sync/ (not full_sync_manager)
    FS/
      worker.py
```

[Michal Rosikiewicz] found that exit code 2 is commented as:

```
# Task failed to start - internal system error
```

### Discrepancy Between Exit Code and Actual Failure Mode

[Erik Andersson]: There is a **mismatch between the exit code and the actual failure mode**. Exit code 2 is labeled "Task failed to start," but in this case the task had successfully started and was running (we know this because it began consuming messages). The task then crashed abruptly with the concurrent map writes error.

> "The task has not failed to start, it has started. It's just that it is crashing horribly when it started."

The exit code mapping may be incorrect or overly broad in the current implementation.

---

## Producer vs. Consumer Error Attribution

### Where Errors Typically Occur

[Erik Andersson]: Errors are **much more likely to occur in the consumer (Audience delivery) side than in the producer (CRM extraction) side**.

The producer rarely fails unless:
- The CRM system has introduced bugs or format changes
- API credentials are incorrect

### Relationship Between Producer and Consumer Lifecycle

[Erik Andersson]: When the consumer fails, it will exit. However, **the behavior when the producer fails is unclear**—Erik acknowledged he needs to verify whether the producer stopping triggers consumer shutdown or if they operate independently.

The existence of a "completed" message from the producer does not guarantee that the consumer has successfully processed all messages.

---

## Full Sync #37838: Audience Consent Export Timeout

### Error Sequence

During the session, a second full sync (ID: 37838) was started at 10:21. Michal and Erik tracked its logs and discovered:

**Producer logs**: No new events received for approximately 1 hour
**Consumer logs**: 
```
Profile export result not ready after 15 minutes
Audience export failed
```

[Michal Rosikiewicz] and Erik identified that the producer had completed its work (marked as "total is known"), but the consumer was stuck waiting for an Audience consent export callback.

### Consent Export Optimization Logic

[Erik Andersson]: When a full sync runs, it performs exports on all topics with configured consent mappings. The flow is:
1. Export existing consent status from Audience
2. Compare with CRM consent data
3. Optimize: if contact already has opt-in in Audience **and** receives opt-in from CRM, skip redundant processing
4. Write any new consent changes

The consent export can fail to trigger in an edge case:

> "There is an annoying case where the Audience export can fail to trigger, but you don't know that unless you have the callback functionality."

### Callback Architecture Limitations

[Erik Andersson]: The Full Sync system **does not have callback functionality for consent exports**. This is by design:

- Full syncs are initiated from the Full Sync Manager
- Manager starts ECS tasks (producer and consumer)
- Audience service would need to send asynchronous callbacks back to these separate ECS tasks
- At the time of building, this architecture was deemed overly complex, so polling with timeout was chosen instead

The downside: if an Audience export silently fails, the sync will timeout after 15 minutes with no clear indication that the export never triggered.

### Timeout Handling

When the 15-minute timeout occurs:
- The sync does not necessarily fail entirely
- The consent export is aborted
- The sync may continue processing other contacts (non-optimal but not catastrophic)

[Erik Andersson]: The sync should still mark "total is known" and proceed, though it will process some consent entries that could have been optimized away.

---

## Debugging Recommendations and Unresolved Questions

### Immediate Action Taken

[Michal Rosikiewicz]: Decided to cancel the running full sync (37838) and start a fresh one to avoid further data inconsistency.

### Items Requiring Further Investigation

[Erik Andersson] identified several areas needing investigation:

1. **Concurrent map writes root cause**: Why is this race condition appearing now after 2 years of unchanged code? Is it related to load, timing, or a recent environmental change?

2. **Full sync "total is known" flag**: Need to verify in the database whether the flag was set correctly for the failed sync, indicating whether the producer properly completed.

3. **Audience export timeout frequency**: Is the 15-minute timeout a one-off or a recurring issue? Does it correlate with specific consent topics or contact volumes?

4. **Exit code 2 mapping**: The exit code system may need revision to distinguish between "failed to start" and "started but crashed."

### Cross-Functional Notes

[Michal Rosikiewicz] noted that the form he was working with involved updating consent on a topic by submitting data to Audience. He suspected a possible relationship between recent consent form changes and the sync failure, but Erik determined the current sync error (Audience export timeout) was not directly caused by the form change, as the same timeout appears in multiple syncs.

---

## Key Takeaways

1. **Three-layer diagnosis model**: Always start with Full Sync Manager logs; if production has started, check Producer logs; if messages are being consumed, check Consumer logs.

2. **Race condition identified**: A "concurrent map writes" error is occurring in the consumer, unusual given the logic hasn't changed in 2 years. Needs root cause analysis before production rollout.

3. **Audience consent export is a weak point**: The 15-minute timeout with no callback acknowledgment can mask silent failures. Consider implementing callback functionality or more robust retry logic.

4. **Exit code taxonomy is imprecise**: Exit code 2 ("task failed to start") does not accurately reflect crashes that occur after startup.

5. **Producer/Consumer independence unclear**: The relationship between producer and consumer failure modes should be documented and verified in the codebase.

6. **Full sync optimization trade-off**: Skipping callback handling for consent exports was a deliberate simplification choice; the cost is visibility into export failures and longer debug cycles.

---

## Unresolved Questions & Action Items

- [ ] **Concurrent map writes**: Root cause analysis needed. Is this a recent regression or a latent race condition?
- [ ] **Database state**: Check if full sync #37838 (and earlier failed sync) has "total is known" flag set correctly.
- [ ] **Audience export timeout**: Is this a recurring pattern or a one-time glitch? Does it correlate with specific consent configurations?
- [ ] **Producer/Consumer failure coupling**: Document and test what happens when producer fails vs. when consumer fails.
- [ ] **Exit code improvement**: Consider updating the exit code mapping or adding detailed exit codes to distinguish failure modes.
- [ ] **Callback architecture revisit**: Evaluate feasibility of implementing callback-based consent export confirmation for better observability.