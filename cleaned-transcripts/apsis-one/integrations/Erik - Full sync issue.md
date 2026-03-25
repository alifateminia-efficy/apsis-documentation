---
source_file: Erik - Full sync issue.txt
domain: Apsis One Integrations
topics: [Full Sync Debugging, Error Logging and Diagnostics, Exit Code Analysis, Producer/Consumer Architecture, Audience Export Timeouts, Race Conditions, Consent Mapping]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, Redis, Audience Service, CRM System, CloudWatch Log Groups]
session_type: debugging-session
subdomains: [Architecture]
---

## Session Overview

This debugging session focused on diagnosing a failed full sync with exit code 2. The team investigated a rare **concurrent map writes** race condition that occurred during consumer processing, explored the multi-layer logging architecture (manager, producer, and consumer logs), traced an audience export timeout issue, and discussed architectural limitations around callback functionality for asynchronous consent exports. The session revealed both a mysterious race condition in unchanged code and an audience service timeout during consent status verification.

---

## Full Sync Architecture and Log Structure

### The Three-Layer Logging Model

[Erik Andersson]: Full syncs have three distinct places where errors can originate, each requiring investigation in different log groups:

1. **Full Sync Manager Log Group** — Where synchronous requests are initiated. Errors here occur when a full sync fails to start (e.g., immediate failure when clicking the "full sync" button).

2. **Producer Log Group** — Part of the actual sync execution. The producer is an ECS task that reads messages from the CRM system and puts them into a temporary queue.

3. **Consumer Log Group** — The other ECS task in the sync execution. The consumer reads messages from the queue and sends them into the Audience service.

### Identifying Which Logs to Check

[Erik Andersson]: When a full sync has begun producing messages (indicated by seeing totals and profile counts), the issue is not in the manager. It has moved into producer and/or consumer territory.

In stage environments, there are typically two log groups for full syncs:
- A consolidated "all events" log group (contains events from both syncs, hard to parse)
- Dedicated log streams for individual syncs (easier to filter and diagnose)

Each log stream in the dedicated log group represents a unique full sync because each sync starts its own ECS task. The log stream timestamp indicates when that particular sync started.

[Michal Rosikiewicz]: In the session, they identified that a sync starting at 10:19 and another at 10:21 were two separate syncs, visible as distinct log streams.

---

## The Concurrent Map Writes Race Condition

### The Error

During consumer processing, a fatal error was encountered:

```
Concurrent map writes
Profile: 26216
```

[Michal Rosikiewicz]: This error occurred during a full sync, meaning all profiles from the CRM were being downloaded. The profile ID 26216 was not something he had explicitly modified, raising the question of whether recent changes caused it.

[Erik Andersson]: Even if you didn't directly update a profile, if a full sync is running, it downloads everything from the CRM, so the profile must be included. The existence of the profile in the CRM is sufficient to trigger the sync.

### Why This Is Unusual

[Erik Andersson]: This concurrent map writes race condition is genuinely puzzling because the logic in question has not changed for approximately 2 years. It's not a frequent occurrence, suggesting either:
- An environmental condition or timing issue
- A recently introduced subtle race condition
- An interaction with other components

The fact that it crashes with `ConcurrentModificationException`-like behavior in stable, unchanged code warrants investigation but is not obviously connected to recent functional changes.

---

## Exit Code Analysis

### Exit Code 2 Definition

The team investigated where exit code 2 is defined in the codebase:

```
Location: /full-sync folder/fs/worker (commented in code)
Definition: Unrecoverable failure
```

[Michal Rosikiewicz]: The code shows exit code 2 is marked as an unrecoverable failure, which aligns with the panic caused by the race condition.

### Exit Code Source Confusion

[Erik Andersson]: The exit code mapping shows:
```
1. Some task failed to start
2. Task stopped (related to unrecoverable failures)
```

However, there's a discrepancy: The task didn't fail to start; it started successfully and then crashed. The "task stopped" code may be giving an incorrect mapping for this scenario where the task exits abruptly after starting.

The UI displays:
```
Exit Code 2 → Internal System Error
```

---

## Producer vs. Consumer Failure Modes

### When Producer Fails

[Erik Andersson]: The producer (which downloads from the CRM system) rarely fails unless:
- The CRM vendor has introduced bugs or accidentally changed the API response format
- API credentials are incorrect

When the producer fails, the consumer will exit because it will not receive messages and the sync cannot complete properly.

### When Consumer Fails

[Erik Andersson]: It's unclear whether the producer stops if the consumer fails. This dependency relationship is not explicitly documented and would need code inspection.

### The Interdependency Question

In the current debugging session, the first sync failed with the concurrent map writes error (consumer-side). The question of whether the producer task was still running or had already completed was not definitively answered during the session.

---

## Identifying Specific Sync Runs

### Using Sync IDs in Logs

To correlate between producer and consumer logs when multiple syncs are running:

1. Get the full sync ID from the UI (displayed in the full sync details)
2. Search for this ID across log groups
3. Producer logs and consumer logs will both contain this ID but in different log streams

In the session:
```
Sync 1 started: 10:19 (ID: 37830something, producer: C4A2...)
Sync 2 started: 10:21 (ID: 37838, producer: F3C4 to IF3C4 to I)
Sync 3 still running: (ID: 37838)
```

[Michal Rosikiewicz]: The distinction is critical because logs can appear out of order due to UTC timezone conversions and AWS log aggregation timing.

---

## Audience Export Timeout Issue

### The Error Message

During the still-running sync (37838), the consumer logs showed:

```
Profile export result not ready after 15 minutes
```

And:

```
Audience exports failed somehow
No [something] filtered export using cat
```

### What This Means

[Erik Andersson]: The full sync performs consent exports for every topic that has a consent mapping set up. The purpose is to:
1. Export the existing consent status from Audience for each topic
2. Compare what Audience has with what we're receiving from the CRM
3. Avoid redundant processing if consent status already matches

If Audience already has an opt-in for a contact on a specific topic, and the CRM also sends an opt-in for that same contact/topic, the sync can skip redundant processing.

### Why the Export Timed Out

[Erik Andersson]: The export timeout happens for one of two reasons:
1. **Audience service is slow** — Taking longer than 15 minutes to respond to the export request
2. **Export trigger failure** — The audience export failed to trigger, but this failure is not visible unless callback functionality exists

### The Callback Functionality Gap

[Erik Andersson]: Full syncs do **not** have callback functionality for consent exports. This is a known architectural limitation:

**Why callbacks weren't implemented:**
```
Triggering: Full Sync Manager → starts ECS tasks (Producer + Consumer)
Callbacks needed: Audience Service → asynchronous requests to two separate ECS tasks
Problem: Complex infrastructure for asynchronous callbacks to ephemeral ECS tasks
Decision made: Not implement callback functionality to avoid architectural complexity
```

Without callbacks, the consumer can only:
- Wait for a timeout (15 minutes)
- Assume the export failed without concrete evidence

With callbacks, the consumer would receive explicit success/failure notification from Audience.

### Impact on Sync Completion

[Erik Andersson]: If an audience consent export times out and is aborted:
- The sync doesn't necessarily fail
- The sync may proceed with "incomplete" audience data
- Consents from the CRM are still processed and written
- The optimization (skipping redundant consent updates) is lost, but data consistency is maintained

However, [Erik Andersson] noted uncertainty about whether the sync should have actually failed at this point, and recommended checking the database to see if the full sync was marked as `total_is_known` (indicating the producer finished and the total number of contacts to sync is known).

---

## Debugging Workflow and Best Practices

### Step-by-Step Diagnostic Approach

1. **Check if full sync started** → Check Full Sync Manager logs (if immediate failure)
2. **Check for message production** → Look for profile counts and totals (indicates producer started)
3. **Narrow down to specific sync** → Use sync ID to search across log groups
4. **Check producer logs** → Look for CRM connectivity issues, API failures
5. **Check consumer logs** → Look for processing errors, audience service issues
6. **Verify sync state in database** → Check `total_is_known` flag to understand sync progress

### Log Group Navigation Tips

- Avoid searching "all events" log groups when multiple syncs are running
- Use dedicated log streams for individual syncs
- Filter by sync ID to correlate producer and consumer behavior
- Account for UTC timestamp conversions when correlating events

---

## Unresolved Questions and Action Items

### Outstanding Issues

1. **Race condition in stable code** — Why is `ConcurrentMapWrites` occurring in logic unchanged for 2 years? Needs investigation into:
   - Whether this is an environmental/timing issue
   - Whether worker thread synchronization has degraded
   - Whether recent changes to calling code introduced unexpected concurrency patterns

2. **Producer-Consumer failure dependency** — Does the producer task stop if the consumer fails? Needs code inspection.

3. **Audience export timeout root cause** — Was the timeout due to slow Audience service or failed export trigger? Currently opaque without callback functionality.

4. **Full sync state in database** — Was the sync marked `total_is_known` before the consumer error? Indicates whether producer fully completed.

### Recommended Next Steps

[Erik Andersson]: 
- Cancel the currently running sync (37838) to stop the timeout
- Check the database for full sync state records
- Investigate the concurrent map writes error in the context of recent changes (even though the code is unchanged, calling patterns may have changed)
- Consider whether callback functionality should be added for consent export visibility

[Michal Rosikiewicz]: 
- Continue testing and reproducing the issue
- Attempt to break the sync in controlled ways to understand failure modes

---

## Key Takeaways

1. **Full sync failures require investigating three log sources**: Manager (startup), Producer (CRM download), Consumer (Audience processing) — check them in order of sync lifecycle.

2. **Producer and Consumer are independent ECS tasks** with different failure modes. Producer fails rarely (CRM issues), consumer fails more often (audience service or data processing issues).

3. **Concurrent map writes in unchanged code is a red flag** — Not a frequent occurrence and suggests either environmental timing issues or subtle race conditions in calling code.

4. **Audience consent export timeouts lack visibility** — Without callback functionality, you can't distinguish between slow service and failed exports. This is a known architectural limitation of the current implementation.

5. **Sync ID is the key to correlation** — When multiple syncs run simultaneously, use the sync ID to find the right log streams across producer and consumer logs.

6. **The 15-minute audience export timeout may not cause sync failure** — Syncs may continue with degraded optimization but intact data integrity. Database state should be checked to confirm actual sync completion status.