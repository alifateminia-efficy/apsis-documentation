---
source_file: Erik from Nov 27th 2025.txt
domain: Apsis One Integrations
topics: [Full Sync Error Diagnosis, CloudWatch Logs Analysis, Producer-Consumer Architecture, Exit Code Handling, Consent Export Timeouts, Concurrent Map Race Conditions]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Producer ECS Task, Consumer ECS Task, CloudWatch Log Groups, Redis, Audience Service, CRM System, Exit Codes]
session_type: debugging-session
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This debugging session focused on troubleshooting a full sync failure that produced exit code 2 and concurrent map write race conditions. Erik walked Michal through the multi-layered logging architecture for full syncs, explaining how to navigate CloudWatch log groups to distinguish between manager, producer, and consumer failures. The team investigated a specific sync that encountered a `ConcurrentModificationException` in profile 26216, followed by an audience export timeout after 15 minutes, and discussed the implications of missing callback functionality for consent exports.

---

## Full Sync Architecture and Log Navigation

### Three-Layer Log Structure

[Erik Andersson]: For full syncs, there are three places you might need to look depending on the nature of the issue:

1. **Full Sync Manager Log Group** — Check here if a full sync fails to start immediately (e.g., when you click "full sync" and it immediately fails)
2. **Producer Logs** — Once the sync has started and begun producing messages (evidenced by counting profiles), failures are likely in the producer or consumer tasks, not the manager
3. **Consumer Logs** — The consumer processes messages from the queue

The key diagnostic is the presence of totals/counts: if you can see that profile counts have been recorded, the manager successfully initiated the sync, so focus on producer/consumer logs instead.

### Producer and Consumer Tasks

[Erik Andersson]: A full sync consists of one **producer ECS task** and one **consumer ECS task**:

- **Producer**: Reads messages from the CRM system and puts them into a temporary queue
- **Consumer**: Reads messages from the queue and sends them into Audience

Either the producer or consumer can fail, and the error could originate from either component. However, [Erik Andersson] noted that the consumer will exit if the producer fails (to avoid a partial sync), but it was uncertain whether the producer stops if the consumer fails.

### Log Stream Filtering in CloudWatch

[Erik Andersson]: When viewing log groups in CloudWatch (especially in stage with multiple syncs running), filter to avoid the "all events" view which aggregates logs from all syncs. Instead:

- Use dedicated log streams for a specific sync
- Each log stream in the full sync log group (not the manager) represents one unique full sync
- Each sync spawns its own ECS task, creating a new log stream with a timestamp
- To isolate a single sync, search the CloudWatch logs by the full sync ID

---

## Exit Code 2 and Error Code Resolution

### Exit Code 2 Definition

[Erik Andersson]: Exit code 2 is set in the code within the full sync worker component, typically defined under:

```
integrations/
├── full_sync/
│   ├── full_sync_manager/
│   └── fs/ (FS folder)
│       └── worker.go (or similar)
```

The exit codes are not directly visible in CloudWatch logs but are returned when the ECS task exits. If you scroll to the absolute bottom of the error logs, you may find the summary code.

### Exit Code vs. Task Stop Code

[Erik Andersson]: There was confusion during debugging about whether exit code 2 comes from "task stop" or another source. Erik discovered that while the task stop message shows "Internal System Error XC 2", the actual exit code 2 may come from a different point in the code. The distinction is important: **the task did not fail to start, it started successfully and then crashed abruptly**. The "task failed to start" error code would be inaccurate in this case.

---

## Concurrent Map Race Condition

### The Issue

[Michal Rosikiewicz]: During the debugging session, a fatal error appeared in the logs:

```
Concurrent map writes
Profile: 26216
```

[Erik Andersson]: This indicates a **race condition** in the concurrent map access. Despite this being strange because the logic has not changed for approximately 2 years, a race condition was occurring during the full sync. The error happens during the initialization phase when the consumer sets up environments, reads configuration, waits for Redis, starts worker threads, and begins reading messages.

### Possible Causes and Caveats

[Erik Andersson]: The race condition is not something that occurs frequently. The error itself suggests that in a full sync (where everything in the CRM is downloaded), even profiles that weren't directly modified can be affected. However, Michal questioned whether recent changes to profile functionality might be related, and Erik clarified that the race condition logic has been stable for years, suggesting the issue may be environmental or load-related rather than code-change-related.

**Warning**: This race condition produced an "unrecoverable failure" panic, causing the entire sync to abort. The exact trigger remains unclear and may warrant deeper investigation in production.

---

## Audience Export Timeout and Consent Handling

### The Timeout Error

[Michal Rosikiewicz]: In the consumer logs, an error appeared:

```
Profile export result not ready after 15 minutes
Audience export failed
Missing consent export
```

This indicates that the Audience service did not respond to an export request within the 15-minute timeout window.

### Two Possible Causes

[Erik Andersson]: An audience export timeout can happen for two reasons:

1. **Audience service takes too long** to respond
2. **Export failed to trigger** — There is an annoying edge case where the audience export can fail silently if you don't have callback functionality

### Lack of Callback Functionality in Full Syncs

[Erik Andersson]: Full syncs currently **do not have callback functionality** for consent exports. This is a design choice made at build time because:

- Full syncs trigger ECS tasks from the full sync manager
- These tasks would need to receive asynchronous callbacks from the Audience service
- The callbacks would need to be routed back to two separate ECS tasks (producer and consumer)
- This architecture was deemed too complex to implement

**Implication**: If an audience export request fails silently, the full sync consumer has no way to know it failed and will timeout waiting for a response. The sync cannot verify that the export actually succeeded.

### Consent Export Optimization and "Aborting"

[Erik Andersson]: During a full sync, exports are performed on all topics that have a consent mapping configured. The process is:

1. Export existing consent status from Audience for comparison
2. Get incoming consent status from CRM
3. **Optimization**: If consent already matches in Audience (e.g., opt-in in both places), skip redundant processing
4. If there's a mismatch, apply the CRM consent status to Audience

When the log says "aborting" an export, it likely means the sync is skipping the consent check for that specific entry (because it's already correct in Audience). This is an optimization, not a failure. **However**, if a consent export truly fails to trigger and never completes, the full sync will still timeout and fail, even though the log message may look like a graceful abort.

[Erik Andersson]: Even if one consent export times out, the sync should not necessarily fail — it's "just one unnecessary processing." However, the full sync marks itself as "total is known" only after the producer finishes. If the consumer times out on a consent export, the full sync status may not be correctly updated in the database.

---

## Identifying Which Sync is Which

### Using Sync IDs and Timestamps

When multiple full syncs are running (e.g., started 2 minutes apart), use the **full sync ID** to correlate logs across producer and consumer:

- Note the full sync ID from the UI (e.g., `37838`)
- Search for this ID in producer logs: `F3C4...` log stream
- Search for the same ID in consumer logs: `C4A2...` log stream
- Cross-reference timestamps, but **note**: CloudWatch may display UTC time, which can be offset from local time by several hours

[Michal Rosikiewicz]: The timestamps in CloudWatch for different log streams may not align perfectly due to AWS time synchronization or UTC conversion, so the full sync ID is a more reliable way to match producer and consumer logs.

---

## Investigation Steps Performed

1. **Started in full sync manager logs** — Confirmed the sync had counted profiles, so manager was not the issue
2. **Moved to producer logs** — Found the producer was running but had no visible errors
3. **Moved to consumer logs** — Found the `concurrent map writes` panic early in initialization
4. **Scrolled to end of consumer logs** — Found exit code 2 and timeout message
5. **Searched both producer and consumer by sync ID** — Confirmed it was the same sync but in separate log streams
6. **Checked for retries** — Found no retries, just immediate abort
7. **Confirmed audience export timeout** — "Profile export result not ready after 15 minutes"

---

## Action Items and Unresolved Questions

### Unresolved

- **What triggered the concurrent map race condition?** The logic hasn't changed in ~2 years, so why is it occurring now? Load spike? Environmental? Interaction with another service?
- **Did the audience export actually fail, or did it just take >15 minutes?** Without callback functionality, the consumer can't distinguish between a real failure and a slow response.
- **Was the "task stop" error code (code 2) the actual exit code, or something else?** Erik was uncertain about the exact source of exit code 2 in the codebase.
- **If the consumer times out, does the producer keep running?** The relationship between producer/consumer lifecycle on failure is unclear.
- **Should the full sync have failed on the consent export timeout, or continued?** Erik suggested it shouldn't necessarily fail for a single export timeout, but the behavior is unclear.

### Recommended Next Steps

[Erik Andersson]:
- Cancel the currently-running sync (sync 37838) to prevent further timeout
- Run another full sync to see if the race condition reproduces
- **Check the database** to see how the failed full sync is marked (specifically, whether "total is known" was set)
- Investigate the concurrent map race condition more deeply (possibly environmental, load, or interaction with recent changes)
- Consider adding callback functionality for consent exports in full syncs (addressed as a future design improvement)

[Michal Rosikiewicz]: Will continue to test and possibly intentionally break the sync again to gather more data on the failure modes.

---

## Key Takeaways

1. **Three-layer log architecture**: Manager logs only show if the sync failed to start. Once started, check producer and consumer logs separately. Use sync IDs to correlate across log streams.

2. **Producer vs. Consumer failures**: Producers rarely fail unless the CRM introduced bugs or credentials are wrong. Most issues are on the consuming side when sending to Audience.

3. **Exit code 2 is defined in code**: The actual definition should be in the full sync worker, not derived from "task stop" messages. The distinction between "failed to start" and "started then crashed" is important for troubleshooting.

4. **Concurrent map writes race condition is a red flag**: Despite stable logic for 2 years, a race condition occurred—environment, load, or timing factors may be involved.

5. **Audience export timeouts can be silent failures**: Without callback functionality, the consumer doesn't know if an export truly failed or just took a long time. This is a known limitation of the current architecture.

6. **Consent export optimization can be confusing**: "Aborting" an export doesn't mean failure—it may mean skipping redundant work. But if an export times out, the full sync still fails.

7. **Database state matters**: Always check the database after a failed sync to see whether "total is known" was set, which indicates whether the producer completed successfully.