---
source_file: Erik from Nov 27th 2025.txt
domain: Apsis One - Integrations
topics: [Full Sync Debugging, Log Analysis, Error Code Handling, Producer-Consumer Architecture, Consent Export Failures, Race Conditions]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, CloudWatch Logs, Redis, CRM System, Audience Service, Consent Mapping]
session_type: debugging-session
---

## Session Overview

This debugging session focused on investigating an exit code 2 failure in a full sync operation. Erik walked Michal through the multi-layered logging architecture for full syncs, identifying where to look based on the failure stage. The session uncovered a concurrent map write race condition in the consumer, a timeout in the audience export callback, and raised questions about error handling and sync state tracking in the database. The conversation highlighted the current limitations of the full sync architecture, particularly around asynchronous callback handling for consent exports.

---

## Full Sync Architecture and Log Locations

### Understanding the Multi-Component System

[Erik Andersson]: Full syncs consist of two separate ECS tasks operating in parallel:
- **Full Sync Manager**: Initiates the full sync. If it fails to start, errors appear in the full sync manager log group.
- **Producer Task**: Reads messages from the CRM system and puts them into a temporary queue
- **Consumer Task**: Reads messages from the queue and sends them to the audience service

The key diagnostic indicator is whether the sync has produced counts/totals. If totals are present, the manager has successfully started and production/consumption has begun, so the manager logs are not the place to investigate.

### Log Group Organization

In stage environments, multiple log groups exist for full sync operations. When investigating a specific sync failure, you should use dedicated log streams rather than "all events" to avoid mixing data from multiple syncs.

[Erik Andersson]: Each full sync operation spawns its own log stream because each one starts its own task. The timestamp of the log stream indicates when that particular sync started. If you started a sync at 10:19 and another at 10:21, the top stream corresponds to 10:19.

The log streams follow a hierarchical structure:
- Full sync manager logs
- Full sync producer logs (separate from manager)
- Full sync consumer logs (separate from producer)

---

## Concurrent Map Write Race Condition

### The Error Signature

During investigation of a full sync failure, the logs revealed:
```
Concurrent map writes - fatal error for profile 26216
```

[Erik Andersson]: This is a race condition in the consumer code. It's particularly odd because this logic has not changed for 2 years. The error occurs during the initialization phase when the consumer is:
- Setting up the environment
- Reading configuration
- Starting worker threads
- Waiting for Redis connectivity
- Beginning message consumption from the queue

### Implications and Uncertainty

[Michal Rosikiewicz]: The race condition was encountered when downloading everything from the CRM system during a full sync. The question arose whether recent code changes related to profile updates could be connected to this failure.

[Erik Andersson]: Even if a profile wasn't directly modified, a full sync downloads everything in the CRM system, so the race condition can manifest regardless. However, the fact that this logic has remained unchanged for 2 years suggests the concurrent map write may be a very rare edge case that doesn't occur frequently.

The root cause of this re-emergence is unknown and requires further investigation.

---

## Exit Code Handling and Task Termination

### Exit Code 2 Definition

[Erik Andersson]: Exit codes are defined in the backend code, not in the logs themselves. Exit code 2 represents an unrecoverable failure/panic.

The exit codes are defined in the full sync integration module:
```
File path: /integration/fs/worker
```

When a task crashes, the exit code may be reported as "task stopped - internal system error: exit code 2" in the ECS service logs.

### Confusion Around Task Failure vs. Abrupt Exit

There was debate about whether the exit code comes from a "task failed to start" condition vs. a task that started successfully but crashed. [Erik Andersson]: The task did not fail to start—it started successfully but then exited very abruptly, crashing when it encountered the concurrent map write race condition. This distinction matters because "task failed to start" would indicate a different failure mode.

---

## Identifying and Diagnosing Specific Syncs

### Using Sync IDs for Log Navigation

When multiple syncs are running, it's critical to correlate them across producer and consumer log groups using the full sync ID. [Michal Rosikiewicz] demonstrated this:
- Sync started at 10:19 (first sync)
- Sync started at 10:21 (second sync)
- Third sync still in progress with ID 37838

[Erik Andersson]: To find a running sync's logs:
1. Get the full sync ID from the UI
2. Search for that ID in the log groups using a 2-hour time window filter
3. This will surface both producer and consumer logs for that specific sync

### Time Zone Considerations

When correlating timestamps, account for UTC timezone offsets. [Michal Rosikiewicz] noted that AWS CloudWatch may report times in UTC while the UI shows local time, which can create confusion when trying to match sync IDs to log streams.

---

## Consent Export Timeout and Audience Service Integration

### The Profile Export Result Timeout

The consumer logs revealed:
```
Profile export result not ready after 15 minutes
```

This indicates that the audience service failed to return consent export data within the expected 15-minute window.

### Two Possible Causes

[Erik Andersson]: This timeout can occur for two reasons:
1. **Audience service is slow**: The audience service is taking longer than 15 minutes to respond
2. **Export failed silently**: The audience export failed to trigger, but there's no way to detect this without callback functionality

### Current Architectural Limitation: Lack of Callback Functionality

The full sync system currently does not implement callback functionality for consent exports. [Erik Andersson]: This was a deliberate architectural decision made at the time of implementation.

> "The full syncs don't have any endpoints to receive callbacks to... you would trigger something from the full sync manager which will start ECS tasks which will then need to get asynchronous requests or asynchronous callbacks from some other service from audience which then needs to go to these two separate ECS tasks. It was not a funny thing to implement."

This means:
- If an audience export fails silently, the full sync has no way to know
- The only detection mechanism is the 15-minute timeout
- This is a known pain point but has not been addressed due to complexity

### Consent Export Optimization Logic

When running a full sync, the system exports consent status from audience for all topics with configured consent mappings. The purpose is optimization: if a contact already has the correct consent status in audience, the sync avoids re-processing it.

However, if the audience export fails (or times out):
- The sync aborts the consent status check for that topic
- It still processes incoming consent data from the CRM
- This is not a failure condition—it's just less efficient (more unnecessary processing)

[Erik Andersson]: The sync should still complete successfully even if the consent export times out, since the consent mappings will still be handled from the CRM data. However, it's unclear whether a timeout should cause the entire sync to fail or just skip that optimization step.

---

## Full Sync State Tracking in Database

[Erik Andersson]: When the producer finishes reading from the CRM, it updates the full sync record in the database with a flag: **total is known**. This indicates that the producer has completed and the system knows the total number of contacts that should be synced.

This flag is critical for understanding sync progress and state. After a problematic sync, it's important to check the database to verify:
- Whether this flag was properly set
- What the current state of the sync is recorded as
- Whether the sync was properly marked as complete or failed

---

## Producer vs. Consumer Failure Modes

### Producer Failure Characteristics

[Erik Andersson]: The producer (CRM data reading stage) very rarely fails unless:
- The CRM system has introduced bugs or accidentally changed the API response format
- Incorrect API credentials are being used

Producer failures are uncommon and usually indicate external system issues rather than problems with the integration code.

### Consumer Failure and Producer Shutdown

If the producer fails, the consumer will exit because it will not receive any more messages and the sync cannot complete properly.

However, there is uncertainty about the reverse: **does the producer stop if the consumer fails?** [Erik Andersson] stated he would need to check this behavior in the code, as it's not immediately clear whether the producer has logic to detect consumer failure and halt.

---

## Recommendation and Next Steps

[Erik Andersson]: For the problematic sync (ID 37838) that was stuck with the audience export timeout:
- Cancel the running sync
- Run another full sync to get a fresh attempt
- Check the database to examine how the previous sync was recorded (whether it's marked as failed, complete, or in some intermediate state)

The concurrent map write race condition and audience export timeout represent two separate issues that both occurred in this debugging session and warrant investigation:
1. Why the race condition re-emerged after 2 years of stability
2. Why the audience export is timing out (slow response or silent failure)

---

## Key Takeaways

1. **Log Navigation Strategy**: Full sync errors can originate from three places (manager, producer, consumer). Determine where to look based on whether the sync has produced counts/totals. Use dedicated log streams filtered by sync ID rather than "all events" when investigating specific syncs.

2. **Rare Race Condition**: The concurrent map write error in the consumer is unexpected given the code hasn't changed in 2 years. This suggests an edge case that surfaces rarely, and the root cause requires investigation.

3. **Timeout vs. Failure Distinction**: A 15-minute timeout on audience consent export doesn't necessarily mean the sync should fail—it may just mean the optimization step is skipped. The current system has no callback mechanism to detect silent failures in audience exports, only timeout detection.

4. **Architectural Limitation**: Async callbacks from audience to full sync ECS tasks were intentionally not implemented due to complexity. This means failures in audience exports can only be detected via timeout, not via immediate feedback.

5. **Exit Code Semantics**: Exit code 2 indicates an unrecoverable panic/crash. This should occur during task runtime, not during task startup, though the error reporting can be ambiguous.

6. **Producer-Consumer Separation**: The two-task architecture provides isolation but creates complexity in error correlation. Always use sync ID and timestamps to match producer and consumer logs.

7. **Database State Verification**: After a failed sync, check the database to see whether the "total is known" flag was set, which indicates how far the producer progressed.

---

## Unresolved Questions and Action Items

1. **Concurrent Map Write Root Cause**: Why has the race condition in the consumer re-emerged after 2 years? Is it related to recent changes, increased concurrency, or a specific input pattern?

2. **Producer Shutdown on Consumer Failure**: Does the producer task halt if the consumer task fails? This behavior is not documented in the current conversation.

3. **Audience Export Timeout Cause**: Is the 15-minute timeout due to slow audience service performance or a silent failure in the export trigger?

4. **Sync Failure Decision**: Should a full sync be marked as failed or partial if the audience consent export times out, or should it be considered a recoverable scenario?

5. **Database State Verification**: Check the database record for sync 37838 to confirm whether it was marked with "total is known" and what its final state is recorded as.