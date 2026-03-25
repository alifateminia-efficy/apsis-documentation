---
source_file: Erik from Nov 27th 2025.txt
domain: Apsis One Integrations
topics: [Full Sync Error Diagnosis, CloudWatch Log Analysis, Producer-Consumer Architecture, Exit Code Handling, Consent Export Failures, Race Conditions]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, Redis, Audience Service, CRM System, CloudWatch Logs]
session_type: debugging-session
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This session focused on diagnosing a full sync failure in the Apsis One integrations platform. Erik walked Michal through the multi-layered approach to identifying where errors occur in full syncs, including analysis of CloudWatch log groups, understanding the producer-consumer ECS task architecture, and investigating an exit code 2 error combined with a concurrent map writes race condition. The team also encountered a timeout issue with audience consent export callbacks and discussed the architectural decision to not implement callback functionality for full syncs.

---

## Full Sync Architecture and Error Location Strategy

### Three-Layer Logging Approach for Full Sync Failures

[Erik Andersson]: Full syncs have three distinct places where errors might manifest, depending on the nature of the issue:

1. **Full Sync Manager Log Group** — Only check here if the full sync fails immediately upon initiation (e.g., user clicks "full sync" and it fails to start)
2. **Producer Logs** — Used when the full sync has started and is actively processing (indicated by message counting and profile totals appearing)
3. **Consumer Logs** — Also relevant once production and consumption have begun

> If you can see that totals have been counted, that means production and consumption have actually started, so the manager logs are not where you should start looking.

### Producer and Consumer Task Structure

A running full sync consists of exactly two ECS tasks:

- **Producer ECS Task**: Reads messages from the CRM system and puts them into a temporary queue
- **Consumer ECS Task**: Reads messages from the queue and sends them to the audience service

[Erik Andersson]: The consumer will exit if the producer fails because we know we will not get a proper sync. However, the behavior when the consumer fails is unclear — the producer may continue running even if the consumer stops.

### Log Stream Organization in CloudWatch

When viewing log groups in stage environments, you will typically have multiple log streams. Instead of checking "all events" (which includes both producer and consumer for all syncs), use dedicated log streams for specific syncs. Each log stream represents one unique full sync, as each sync starts its own task.

> If you have started one now that is running, it's most likely the top one.

You can identify which log stream belongs to which sync by matching the start timestamps and full sync IDs.

---

## Concurrent Map Writes Race Condition and Exit Code 2

### The Error Observed

During the debugging session, logs revealed a **fatal error: concurrent map writes** for a specific profile (ID 26216). This occurred despite the underlying logic not changing for 2 years.

[Erik Andersson]: This concurrent map writes error indicates a race condition, which is strange because this logic has not changed for a long time. I honestly have no idea what would cause this or whether it occurs frequently.

### Exit Code 2 Investigation

Exit code 2 indicates an unrecoverable failure, specifically an internal system error. The code location was traced as follows:

- Exit codes are defined in the backend code
- They will not appear in the full sync logs themselves
- The code is set in the `fullsync/worker` module

[Erik Andersson]: The task stop is giving an incorrect code because the task didn't fail to start — it started and then crashed horribly afterward. The concurrent map writes panic is what caused the abrupt exit.

**Note**: The exact mechanism for how exit code 2 is surfaced in the UI (versus remaining internal to logs) requires further investigation.

---

## Full Sync Database State: "Total is Known" Flag

When the producing part of a full sync completes successfully, the system updates the full sync record in the database with a **"total is known"** flag. This signals that:

- The producer has finished reading from the CRM
- The system knows the total number of contacts that should be processed
- The consumer can now proceed with message consumption

[Erik Andersson]: After a full sync producer completes, we should verify in the database whether the full sync is marked with "total is known." This is a critical checkpoint for understanding whether the producer truly finished or if it crashed mid-way.

---

## Audience Consent Export and Callback Architecture

### Consent Export Timeout Issue

During the full sync, the system performs exports of existing consent status from Audience for all topics with configured consent mappings. This is an optimization: if a contact already has opt-in consent in Audience for a topic, and the CRM also provides opt-in, the system skips processing that consent entry to avoid unnecessary work.

In the debugging session, a consent export timed out with the message:

```
Profile export result not ready after 15 minutes
```

[Michal Rosikiewicz]: The error showed "profile export result not ready after 15 minutes," indicating the audience export didn't respond in time.

[Erik Andersson]: This timeout can happen for two reasons:
1. Audience takes too long to respond
2. Something went wrong with the export, and there's no way to know unless you have callback functionality

### Callback Functionality Limitation

[Erik Andersson]: Full syncs don't have callback functionality for consent exports. This was an intentional architectural decision made during initial implementation. The reason: full syncs are triggered from the manager and spawn separate ECS tasks that would need to receive asynchronous callbacks from the Audience service. This setup would be complex because requests from Audience would need to reach two separate ECS tasks running independently.

Instead of implementing bidirectional async callbacks, the system uses polling with a 15-minute timeout. If no response is received in 15 minutes, the export is marked as "not ready."

> When you handle a timeout for a consent export, the sync doesn't necessarily fail — it simply stops checking that particular consent status. If you receive an opt-in for a contact on that topic from the CRM, you still process it, even if you couldn't verify the existing Audience consent. It's just unnecessary processing, but not a failure.

---

## Debugging Log Navigation Tips

### UTC Timezone and Timestamp Matching

When searching CloudWatch logs, remember that timestamps are in UTC. If the sync was triggered at 10:21 AM local time, the logs will show a different hour depending on your timezone offset.

[Michal Rosikiewicz]: There was a discrepancy where logs showed timestamps that didn't match the local trigger time. Erik clarified that this was due to UTC conversion — local time was 6 hours ahead of the logged UTC time.

### Searching by Full Sync ID

To find logs for a specific running sync:
1. Get the full sync ID from the UI (e.g., `37838`)
2. Search across both producer and consumer log groups with a 2-hour time window
3. Match the sync ID in the log stream names or within the log content

[Erik Andersson]: The producer and consumer are separate log groups. You were looking at the consumer; I was looking at the producer — that's why I couldn't see it.

---

## Recommended Actions After Full Sync Failure

1. **Cancel the failing sync** to prevent further resource consumption and unnecessary Audience API calls
2. **Check the database** to verify whether the full sync is marked with "total is known"
3. **Review both producer and consumer logs** separately to pinpoint which component actually failed
4. **Run another full sync** after identifying and resolving the root cause

---

## Key Takeaways

1. **Three-layer troubleshooting**: Use Full Sync Manager logs only for startup failures; use Producer/Consumer logs once the sync is running. Identify which layer failed by checking whether message production/consumption has begun.

2. **Race condition encountered**: A concurrent map writes error appeared in logs despite unchanged code for 2 years. The cause is unknown and requires further investigation. This caused exit code 2 (unrecoverable failure/panic).

3. **Producer-Consumer separation**: The architecture uses two independent ECS tasks. The consumer exits if the producer fails, but the producer's behavior when the consumer fails is unclear and warrants investigation.

4. **Exit code 2 handling**: Exit codes are defined in backend code, not visible in logs directly. Exit code 2 indicates an internal system error / panic. The system may incorrectly report "task failed to start" when the task actually started but crashed.

5. **Consent export optimization has limits**: The system optimizes by skipping consent processing if it already exists in Audience, but without callbacks, timeouts cannot distinguish between slow responses and genuine failures. The sync continues anyway, just with potentially redundant processing.

6. **Callback architecture trade-off**: Full syncs deliberately omit callback functionality for consent exports to avoid the complexity of routing asynchronous responses from Audience back to independent ECS tasks. This resulted in a simpler implementation but less visibility into export failures.

7. **Database state tracking**: Always check the "total is known" flag in the database after a full sync completes or fails to understand where in the pipeline the failure occurred.

---

## Unresolved Questions

- What causes the concurrent map writes race condition after 2+ years of unchanged code? Is it intermittent or reproducible?
- Does the producer continue running if the consumer fails, or does it also exit? This behavior is unclear.
- How exactly is exit code 2 surfaced in the UI versus remaining internal to ECS task logs?
- Could the recent changes to profile update functionality be related to the race condition, or is it unrelated?
- Should the full sync fail when a consent export times out, or is it acceptable to continue with redundant processing?