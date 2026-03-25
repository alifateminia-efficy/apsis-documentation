---
source_file: Erik - Full sync issue.txt
domain: Apsis One Integrations
topics: [Full Sync Error Diagnosis, Exit Codes, Log Analysis, Producer-Consumer Architecture, Race Conditions, Consent Export Timeouts, AWS CloudWatch Logging]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Producer ECS Task, Consumer ECS Task, CRM System, Audience Service, Redis, CloudWatch Log Groups, Database]
session_type: debugging-session
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This debugging session focused on diagnosing a failed full sync operation that exited with error code 2. The discussion covered how to navigate CloudWatch log groups to identify where failures occur (manager vs. producer vs. consumer), traced a concurrent map write race condition in the consumer, and uncovered a timeout issue with audience consent export functionality. The team explored the producer-consumer architecture of full syncs, the exit code handling in the codebase, and identified that the audience export callback mechanism was not implemented for full syncs.

---

## Full Sync Architecture and Error Diagnostic Flow

### Three-Tier Logging Structure

When diagnosing full sync failures, there are three distinct places to check depending on the nature of the issue:

1. **Full Sync Manager Log Group** — Check this if the full sync fails to start immediately (e.g., user clicks "full sync" and it fails right away). The manager handles synchronous requests that initiate the full sync.

2. **Producer Logs** — Once the full sync has started and is producing messages (evidenced by profile counts appearing), the issue typically lies here or in the consumer, not the manager.

3. **Consumer Logs** — Handles message consumption and delivery to audience service.

[Erik Andersson]: > "For full syncs, there are three places you might need to look in the error for errors depending on the nature of the issue. So you have the full sync manager... If a full sync fails to start, like you click on full sync and it immediately fails. Then it would be in the full sync manager log group you should check."

### Identifying Individual Sync Log Streams

In environments with multiple syncs running (e.g., staging), CloudWatch Log Groups for full syncs contain multiple log streams, with each stream representing one unique full sync instance. Each stream begins with a timestamp indicating when that particular sync started.

[Erik Andersson]: > "Each one starts its own task. So if you have started one now that is running, that's most likely the top one."

When troubleshooting a specific sync, use dedicated log streams rather than the "all events" log group, which includes messages from all syncs and is harder to parse.

---

## Producer-Consumer Task Architecture

Full syncs consist of two separate ECS tasks:

- **Producer Task**: Reads messages from the CRM system and places them into a temporary queue.
- **Consumer Task**: Reads messages from the queue and sends them to the Audience service.

These tasks are independent; if the consumer fails, it exits. However, the behavior of the producer when the consumer fails was unclear during the session and requires further investigation.

[Erik Andersson]: > "The full sync consists of 1 producer ECS task and one consumer ECS task. So the producer reads the messages from the CRM system and puts it into a temporary queue and the consumer reads these messages and and send them into audience."

---

## Concurrent Map Write Race Condition

### The Error

A **fatal error** occurred during profile processing:

```
Concurrent map writes
Profile: 26216
```

This manifested as a panic in the consumer, causing abrupt task termination.

### Analysis

[Erik Andersson]: > "This concurrent map writes. It's there is seems to be some race condition going on, which is weird because this logic has not changed for like 2 years."

The race condition is unusual because the code responsible has remained unchanged for approximately two years. The error did not occur frequently, and the exact trigger was not immediately apparent from the logs. This suggests either:

- A rare, timing-sensitive condition that hadn't been hit before
- An environmental factor or upstream change in the CRM data

[Michal Rosikiewicz]: > "I doubt that there may be some connection with what we changed here and why we cannot update this for example, but it's not this case."

The race condition was deemed unlikely to be related to recent profile-update functionality changes.

---

## Exit Code 2 and Error Code Mapping

### Exit Code Definition

Exit code **2** is defined in the codebase (specifically in the full sync worker module) as an **unrecoverable failure**, typically corresponding to a panic or crash scenario.

[Erik Andersson]: > "Obviously this is a panic because we've run into some weird race condition which we can't recover from it."

### Where Exit Codes Are Defined

Exit codes are set in the codebase, not in the logs directly. The relevant code is located in:

```
integrations/fullsync/worker.py
```

(Or similar path depending on the language/structure—the exact file was being located during the session.)

### UI-Level Error Mapping

When the exit code 2 is reported in the Apsis One UI, it appears as:

```
Internal System Error (XC 2)
```

The "task stopped" error message can be misleading because it implies the task failed to start, when in reality the task started but crashed internally during execution.

[Erik Andersson]: > "The task stop is giving the incorrect code for it because the task has not failed to start, it has started. It's just that it is crashing horribly when when it started."

---

## Audience Service Consent Export Timeout Issue

### The Problem

During a full sync for profile 37838, the consumer logs showed:

```
Profile export result not ready after 15 minutes
Audience export failed
```

The audience consent export timed out, preventing the consumer from completing the full sync even though the producer had finished.

### Root Cause: Missing Callback Mechanism

Full syncs do **not** implement callback functionality for consent exports. This was a deliberate architectural decision at build time.

[Erik Andersson]: > "The full sync doesn't have the callback back functionality for the consent exports because like the full syncs don't have any endpoints to receive callbacks to and at least at the time of the building we opted to not have that because like you would trigger something from the full sync manager which will start ECS tasks which will then need to get asynchronous requests or asynchronous callbacks from some other service from audience which then needs to go to these two separate ECS tasks."

Instead of waiting for callbacks, the sync implementation relies on polling with a hardcoded timeout (15 minutes in this case). If the audience service doesn't respond within the timeout window, the export is assumed to have failed or been lost.

### Why This Happens

When running a full sync, the system exports consent status from audience for all topics that have consent mapping configured. This is done to compare existing consent in audience with incoming consent from the CRM, to avoid redundant processing:

- **Optimization**: If a consent state already exists in audience (e.g., opt-in) and the CRM is sending the same state (opt-in), the sync skips processing that entry.
- **Problem**: If the audience export fails to trigger or is lost in transit, the sync stalls waiting for a response that will never come.

[Erik Andersson]: > "We like we export the existing consent status in from audience to compare what we are getting in. So if you have a opt in in audience and we are receiving an opt in from the CRM system, we will not handle that specific consent entry. But... It's just one unnecessary processing, but."

### Mitigation and "Aborting" Behavior

When an audience export times out, the sync logs show "aborting" for that consent check. This does **not** mean the entire sync fails—it means the optimization is skipped for that particular topic/consent entry.

[Erik Andersson]: > "If we get an opt-in for the contact for that specific topic, we still handle that opt-in. So we write the opt-in even if there were to be an opt in inside of audience like because like this is an optimization part, we don't want to handle the consent if it is already in audience like that, that's an unnecessary processing."

The sync will continue processing, resulting in potentially redundant consent updates being sent to audience, but no data loss.

---

## Database State Tracking

### "Total Is Known" Flag

In the full sync producer, once the producing phase completes, the system updates the full sync record in the database with a **"total is known"** flag. This indicates:

- The producer has finished reading from the CRM
- The total count of contacts to be synced is now known and stored in the database

[Erik Andersson]: > "When the producing part is done, we update the full sync in the database with like total is known, meaning that the the producer has finished like we know how many contacts should should be in the sink."

This flag is critical for monitoring the sync progress and determining whether the producer completed successfully or was interrupted.

---

## Debugging Workflow and Log Navigation

### Timestamp Alignment Issues

When navigating CloudWatch logs in AWS, timestamps may be displayed in UTC, which can differ from local time. The session encountered some confusion around log timestamps:

- Producer logs showed events at 18:00 UTC (corresponding to a 10:21 AM local start time)
- The "last event time" shown in log stream metadata reflects the absolute most recent log entry, which may be much earlier if the producer has finished but the consumer is still running

[Michal Rosikiewicz]: > "But it still this is 36 and it it happened like 21... But maybe there was some makeup in AWS."

### Searching Log Streams

To locate a specific sync's logs:

1. Obtain the **full sync ID** from the UI
2. Search CloudWatch log streams using this ID
3. Remember that producer and consumer have **separate log groups** — you must check both to get the full picture

The producer and consumer logs for the same sync may show different behaviors. A consumer error (e.g., timeout) does not necessarily indicate a producer error, and vice versa.

---

## Known Limitations and Workarounds

### Absence of Callback Infrastructure

Because full syncs run in isolated ECS tasks without persistent endpoints, implementing asynchronous callbacks from audience would require:

- A callback endpoint on the ECS tasks
- Task networking complexity to receive callbacks
- Potential coordination between separate instances

This was deemed not worth the complexity at the time of implementation. The trade-off is that the sync relies on polling with fixed timeouts, which can mask silent failures in the audience export mechanism.

[Erik Andersson]: > "It was not a funny thing... [but the architecture would be complex and] we opted to not have that."

### No Retry Logic for Failed Exports

The session revealed that when an audience export times out, there is **no automatic retry**. Once the 15-minute timeout is exceeded, the sync aborts that particular consent check and continues.

[Michal Rosikiewicz]: "Has there been any retries here? No."

---

## Recommendations for Continued Investigation

1. **Check database state** for full sync 37838 to verify whether "total is known" was set and what the expected vs. actual counts are.

2. **Investigate the concurrent map write race condition** — This may require:
   - Enabling additional tracing/debugging for this specific sync
   - Examining the exact CRM data in profile 26216 to see if it has unusual characteristics
   - Considering whether environmental factors (e.g., high load) trigger the race condition

3. **Review audience service export latency** — The 15-minute timeout suggests audience exports are sometimes slow. Consider whether:
   - Audience service performance has degraded
   - The export payload size has increased
   - There are known slowdowns for certain consent topic configurations

4. **Consider implementing callback functionality** — A future enhancement to make the full sync more robust would be to implement proper callback handling, allowing the sync to respond immediately when audience exports complete rather than polling with fixed timeouts.

---

## Key Takeaways

- **Full sync errors have three diagnostic locations**: manager logs (startup failures), producer logs (CRM read errors), and consumer logs (processing/audience errors).

- **Producer and consumer are separate ECS tasks** with independent lifecycles and log streams; both must be checked to diagnose a full sync failure.

- **Exit code 2 = unrecoverable failure/panic** — When this appears in the UI as "Internal System Error (XC 2)", it typically indicates a crash (like the concurrent map write race condition observed), not a startup failure.

- **Audience consent export timeouts are a known limitation** — There is no callback mechanism for audience exports, so the sync polls with a 15-minute timeout. If audience doesn't respond, the sync aborts that particular consent check and continues, potentially sending redundant consent updates.

- **The concurrent map write error is unusual and long-standing** — The race condition logic hasn't changed in 2+ years, suggesting either a rare timing issue or a data/environment factor not yet identified.

- **Canceling a stalled sync and running a fresh one is a reasonable workaround** — This allows the producer to be recreated and potentially avoid the race condition or timeout issue.

---

## Unresolved Questions and Action Items

1. **What triggers the concurrent map write race condition?** — This needs deeper investigation. Is it profile-specific? Load-dependent? CRM-data-dependent?

2. **Does the producer exit when the consumer fails?** — [Erik Andersson] stated uncertainty about this; the behavior should be verified in the codebase.

3. **Why did the audience consent export timeout on this particular sync?** — Was it a transient audience service issue or a systematic problem? Requires investigation of audience service logs.

4. **Should "total is known" have been set for sync 37838 before it was canceled?** — This should be verified in the database to confirm the producer completed.

5. **Is the 15-minute timeout for audience exports configurable, and is it ever exceeded in production?** — Worth reviewing for production readiness.