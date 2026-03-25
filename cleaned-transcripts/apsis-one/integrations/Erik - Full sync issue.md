---
source_file: Erik - Full sync issue.txt
domain: Apsis One Integrations
topics: [Full Sync Error Diagnosis, Exit Code 2, Producer-Consumer Architecture, Concurrent Map Write Race Condition, Audience Export Timeout, CloudWatch Log Navigation]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, Redis, CloudWatch Log Groups, Audience Service, CRM System]
session_type: debugging-session
subdomains: [Outbound Flow, Architecture]
---

## Session Overview

Erik and Michal debugged a full sync failure that exited with code 2. The session covered the three-tier logging strategy for diagnosing full sync issues (Manager → Producer → Consumer logs), identified a concurrent map write race condition in the consumer that caused the initial sync to crash, and discovered a subsequent audience export timeout in a second sync attempt. The session demonstrates the architecture of full syncs (separate ECS tasks for producer reading from CRM and consumer writing to Audience), troubleshooting methodology using CloudWatch log streams, and limitations of the current callback mechanism for consent exports.

---

## Full Sync Architecture and Logging Strategy

### Three-Tier Logging for Full Sync Diagnostics

Full sync issues can originate in three different places, requiring a specific diagnostic approach:

1. **Full Sync Manager Log Group** — Check here only if the full sync fails to start immediately (e.g., you click "full sync" and it fails instantly). The manager is responsible for synchronous requests and initiating the full sync.

2. **Full Sync Producer Logs** — Check here if the sync has started and begun producing messages (indicating totals have been counted). The producer reads messages from the CRM system and puts them into a temporary queue.

3. **Full Sync Consumer Logs** — Check here if production and consumption have both started. The consumer reads messages from the queue and sends them into Audience.

[Erik Andersson]: "If a full sync fails to start, like you click on full sync and it immediately fails, then it would be in the full sync manager log group you should check. But in this case we can see that it has started producing messages so it would not be in the manager we need to check."

### Producer-Consumer Task Architecture

Each full sync consists of exactly **one producer ECS task** and **one consumer ECS task**. They are separate, independent tasks that can fail independently of each other:

- The **producer** reads from the CRM system and places messages into a temporary queue
- The **consumer** reads from that queue and sends data into Audience
- The consumer will exit if the producer fails (since no proper sync can complete without producer data)
- Current behavior unclear: whether the producer stops if the consumer fails

### CloudWatch Log Navigation Best Practices

In stage environments with multiple concurrent syncs, use dedicated log streams rather than "all events" log groups:

- Go to the full sync log group (not the manager), which shows individual log streams
- Each log stream represents one unique full sync run, as each sync starts its own ECS task
- The most recent sync will typically be at the top
- Timestamp matching (start times like 10:19 vs 10:21) helps identify which log stream corresponds to which sync

---

## Concurrent Map Write Race Condition in Consumer

### The Error

During the first full sync attempt, the consumer logs showed:

```
Concurrent map writes
Fatal error encountered for profile 26216
```

[Michal Rosikiewicz]: "This is the fatal error we encountered for the profile 26216, which to be honest I don't know and at least I didn't update it 2/1."

The profile 26216 being unknown is irrelevant — during a full sync, the system downloads everything from the CRM, so all profiles (whether recently modified or not) are included.

### Investigation Findings

- This appears to be a **race condition in concurrent map operations**
- [Erik Andersson]: "This concurrent map writes. It's there is seems to be some race condition going on, which is weird because this logic has not changed for like 2 years."
- The error is a **panic/unrecoverable failure**, not a handled exception
- The frequency of occurrence is unknown (suspected to be rare)
- **Root cause not determined during the session** — this warrants further investigation

### Exit Code Behavior

The concurrent map write error triggered an **exit code 2**, which maps to **"Task stopped: Internal system error"** in the ECS task definitions. However, [Erik Andersson] noted a potential mismatch: the task didn't fail to start; it started correctly and then crashed, so the exit code mapping may be misleading.

Exit code definitions are located in:
```
integration/fs/worker.go
```

---

## Audience Export Timeout Issue

### The Problem in the Second Sync

The second full sync (started 2 minutes after the first, at 10:21) encountered a different failure:

```
Profile export result not ready after 15 minutes
```

This occurred in the **producer logs** when attempting to export consent data from Audience.

[Michal Rosikiewicz]: "So audience exports failed somehow."

### Why This Timeout Occurs

Two possible causes:

1. **Audience service is slow** — took longer than expected to respond
2. **Export failed to trigger silently** — the export request was never initiated or failed without proper notification

[Erik Andersson]: "There is an annoying case where like the audience export can fail to trigger, but you don't know that unless you have like the callback functionality and the full sync."

### Why Callback Mechanism Wasn't Implemented

Full syncs **do not have callback functionality** for consent exports. The reason:

- Full syncs are initiated from the Full Sync Manager, which starts separate ECS tasks
- These ECS tasks would need to receive asynchronous callbacks from Audience
- Audience callbacks would need to be routed back to two separate ECS tasks
- This architecture was deemed too complex at the time of implementation

[Erik Andersson]: "You would trigger something from the full sync manager which will start ECS tasks which will then need to get asynchronous requests or asynchronous callbacks from some other service from audience which then needs to go to these two separate ECS tasks. It was not a funny thing."

### Export Failure Impact: Optimization, Not Critical

During full sync, the system exports existing consent status from Audience to compare with incoming data from the CRM:

- **Scenario**: If Audience shows opt-in for a topic AND the CRM also provides opt-in, the system skips processing to avoid unnecessary work
- **When export fails**: The system aborts that optimization and still processes the consent record
- [Erik Andersson]: "We just don't check that consent status in audience. So if we get an opt-in for the contact for that specific topic, we still handle that opt-in... it's just one unnecessary processing"
- **Conclusion**: The export timeout itself should not cause the full sync to fail, but current behavior is uncertain

### Database State Concern

When an export times out, the full sync should ideally mark **`total_is_known`** in the database once the producer finishes. This flag indicates that the producer has completed and the system knows how many contacts should be processed. This flag's status needed to be verified but was not checked during the session.

---

## Debugging Session Actions and Decisions

### Identified Confusions During Investigation

The team tracked two different sync attempts simultaneously:

- **First sync** (started 10:19): Hit concurrent map write error, producer completed (exit code 2), consumer crashed
- **Second sync** (started 10:21): Producer running, audience export timed out, still in progress at session end

Log stream identification was complicated by:
- UUID naming conventions (e.g., `F3C4A2`, `3783805`, etc.)
- UTC vs local time discrepancies in log timestamps
- Two separate log streams (one for producer, one for consumer) per sync

[Michal Rosikiewicz]: "So we have two different log groups then... you are looking at the producer. Because that happens. Yeah, yeah, OK, now, now I understand why I don't see it."

### Recommended Next Steps

[Erik Andersson]: "I would recommend that you cancel it for now and run another one. But we should check in the database to like check how it looks like."

Specific database checks needed:
- Verify `total_is_known` status in the full sync record
- Determine if the concurrent map write is reproducible
- Check audience export timeout frequency and patterns

---

## Key Takeaways

1. **Three-tier diagnostic approach**: Manager logs → Producer logs → Consumer logs. Only check each level when relevant based on whether the sync started and progressed.

2. **Concurrent map write race condition** is a serious anomaly (not seen in 2 years of codebase) that caused unrecoverable panic in the consumer. Root cause unknown and requires deeper investigation.

3. **Audience export timeout** (15-minute threshold) is non-critical from a functional perspective since the sync will process the consent record anyway. However, the absence of callback mechanisms means failures may not be properly detected or alerted.

4. **Exit code 2 ("Task stopped: Internal system error")** maps to multiple failure modes; the code itself may not accurately reflect whether the task failed to start vs. crashed during execution.

5. **Full sync logs require careful navigation** in multi-sync environments. Use dedicated log streams filtered by sync ID rather than "all events" groups to avoid noise.

6. **Producer-consumer separation** is architectural and intentional. When the producer finishes, it should mark `total_is_known` in the database, signaling the consumer about expected message count.

---

## Unresolved Questions and Action Items

### Questions for Future Investigation

- Why did the concurrent map write race condition occur after 2 years of stable code? Was there a recent deployment or CRM system change?
- Was profile 26216 special in any way, or is the race condition truly data-independent?
- Does the producer stop if the consumer fails, or does it continue producing messages indefinitely?
- What is the actual impact of the audience export timeout on the sync outcome? Should it cause the sync to fail?
- How often does the audience export fail to trigger silently without callback detection?

### Actionable Items

- [ ] **Cancel the current in-progress full sync** (sync ID 37838, running since 10:21)
- [ ] **Check database** for the full sync record status, specifically `total_is_known` flag
- [ ] **Investigate root cause** of concurrent map write race condition — check for recent code changes, CRM API changes, or profile-specific data anomalies
- [ ] **Consider implementing callback functionality** for audience export failures to improve observability
- [ ] **Review exit code mapping** to ensure code 2 accurately reflects task crash scenarios
- [ ] **Run another full sync** to determine if the concurrent map write is reproducible or a one-off anomaly