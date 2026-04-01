---
source_file: Erik - Full sync issue.txt
domain: Apsis One Integrations
topics: [full sync architecture, ECS task structure, error diagnosis, log navigation in AWS CloudWatch, exit codes, concurrent map write panic, audience export timeout, consent optimization logic]
speakers: ["Erik Andersson (senior engineer / domain expert)", "Michal Rosikiewicz (engineer, screen-sharing)", "Tomasz Kowalski (observer)"]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, Audience Service, ECS Tasks, CloudWatch Log Groups, Redis, CRM integration]
session_type: debugging-session
---

# Full Sync Debugging Session — Exit Code 2 and Concurrent Map Write Panic

## Session Overview

This session is a live debugging walkthrough led by Erik Andersson, with Michal Rosikiewicz sharing their screen, investigating a failed full sync on a stage environment. Two separate issues were encountered: a `concurrent map writes` fatal panic in the full sync consumer, and an audience export timeout causing the producer to abort consent pre-fetching. Erik explains the full sync architecture, the correct CloudWatch log groups to consult at each stage of diagnosis, and the rationale behind several design decisions including why the full sync does not use callbacks for consent exports. The session ends with a recommendation to cancel the stuck sync and re-run it, with a follow-up action to inspect the database state.

---

## Full Sync Architecture Overview

### Three-Component Structure

A full sync consists of three distinct components, each with its own log group:

1. **Full Sync Manager** — handles synchronous requests and is responsible for *starting* a full sync. If a full sync fails immediately upon clicking "start" (before any messages are produced or consumed), this is where to look.

2. **Full Sync Producer** (ECS Task) — reads contact/profile messages from the CRM system and puts them into a temporary queue.

3. **Full Sync Consumer** (ECS Task) — reads messages from the temporary queue and sends them into Audience.

> "The producer reads the messages from the CRM system and puts it into a temporary queue and the consumer reads these messages and sends them into Audience."

### How to Determine Which Component to Investigate

- If the sync **fails immediately** (no totals, no profiles counted): check the **Full Sync Manager** log group.
- If the sync has **started producing messages** (totals are visible, profile count has begun): the issue is in either the **Producer** or **Consumer**. Do not look at the Full Sync Manager.

> "It had totals. That means that production and consumption has actually started. So the manager — it's not where we should start looking."

---

## Navigating CloudWatch Log Groups for Full Sync Debugging

### Log Group Structure

- There is a dedicated log group for full syncs with individual **log streams**, where each log stream corresponds to one unique full sync run (one ECS task invocation).
- Avoid using the "All Events" view when there are multiple syncs, as it merges logs from different runs. Instead, navigate directly into the specific log stream.

### Identifying the Correct Log Stream

- Each running full sync has a unique **full sync ID** (e.g., `37838`). Use this ID to search across log groups to find the correct log stream.
- In the session, the sync ID was found via the full sync first-step UI, then used to search CloudWatch with a 2-hour custom time window.

### Producer vs. Consumer Log Streams

- The producer and consumer are **two separate ECS tasks** and therefore appear as **separate log streams** (potentially in the same log group or different groups).
- Example log stream identifiers observed in session:
  - Producer: `F3C4...` (stream for sync `37838`)
  - Consumer: `C4A2...` (stream for sync `37838`)
- **Important:** When Erik and Michal were confused about seeing different content, it was because Erik was in the consumer log stream and Michal was in the producer log stream for the same sync ID.

### Timestamp Note

Log timestamps in CloudWatch are in **UTC**. Be aware of the offset when correlating with local timestamps.

---

## Issue 1 — Concurrent Map Writes Fatal Panic (Consumer)

### Observed Error

At the top of the consumer log stream, a fatal error was logged:

```
concurrent map writes
fatal error encountered for profile 26216
```

This caused an unrecoverable panic and abrupt process exit.

### Log Sequence

The consumer log showed the normal startup sequence:
1. Environment setup
2. Redis connection wait
3. Worker threads starting
4. Consumer begins reading messages

Then the `concurrent map writes` panic occurred mid-processing.

### Exit Code 2

- The exit code `2` visible in the backend/UI is defined in the codebase under:

```
integrations repo > full sync folder (FS folder, above Full Sync Manager) > worker
```

- The exit code is mapped to **"internal system error"** / **"unrecoverable failure"**.
- The code path that surfaces this in the UI goes through **task stopped** handling. Erik noted this is slightly misleading because the task *did* start successfully — it crashed after starting, but the task-stop handler reports it similarly to a task that failed to start.

> "The task has not failed to start, it has started. It's just that it is crashing horribly when it started."

### Assessment

- The `concurrent map writes` panic indicates a **race condition** in the consumer.
- Erik noted this logic has not changed in approximately **2 years** and this type of error is not expected to occur frequently.
- The specific profile (`26216`) involved is not necessarily significant — in a full sync, all profiles in the CRM are downloaded regardless of recent user activity.

> "This logic has not changed for like 2 years... I honestly have no idea what this would be. For sure it's not something that would occur frequently."

⚠️ **Unresolved:** Root cause of the race condition is unknown and warrants further investigation.

### Consumer/Producer Interdependency on Failure

- If the **producer** fails, the **consumer** will exit (it knows a proper sync cannot complete).
- However, Erik was **uncertain** whether the producer stops if the consumer fails.

> "I honestly don't know if the producer stops if the consumer fails."

⚠️ **Unresolved:** Behavior of the producer when the consumer crashes needs to be confirmed.

---

## Issue 2 — Audience Export Timeout (Producer)

### Observed Error in Producer Log Stream

```
Profile export result not ready after 15 minutes
```

Followed by log lines indicating: `missing, missing, missing, missing` and then an abort/timeout on a consent export using a CAT filter.

### What This Export Does

When the full sync producer runs, it **pre-fetches existing consent status from Audience** by triggering exports for all topics that have a consent mapping configured. The purpose is an **optimization**:

- Compare incoming consent data from the CRM against what already exists in Audience.
- If a contact already has an opt-in in Audience and is also sending an opt-in from the CRM, skip processing that consent entry (unnecessary write).

### What Happens When the Export Times Out

If the Audience export does not respond within 15 minutes, the producer **aborts the pre-fetch** for that consent topic. Critically:

> "There is nothing bad happening if we handle that opt-in from the CRM system. It's just one unnecessary processing."

- The sync **continues** and will write the opt-in from the CRM even if one already exists in Audience.
- No data loss or corruption — just some redundant consent writes.
- Erik expressed uncertainty about whether this timeout *should* cause the sync to be marked as failed, and leaned toward thinking it should not.

### Why There Is No Callback Mechanism for Consent Exports in Full Sync

This is a known architectural limitation with documented rationale:

> "The full syncs don't have any endpoints to receive callbacks to. At the time of building we opted to not have that because you would trigger something from the Full Sync Manager which will start ECS tasks which will then need to get asynchronous callbacks from some other service — from Audience — which then needs to go to these two separate ECS tasks. It was not really a funny thing to implement."

The polling-with-timeout approach was chosen over implementing an async callback architecture into ephemeral ECS tasks.

### Why This Timeout Occurred

Two possible causes identified:
1. Audience service took too long to respond (performance issue).
2. The Audience export failed to trigger, but the full sync has no callback to detect this failure — it just waits until timeout.

---

## Full Sync Producer Completion and Database State

When the producing part finishes, the full sync record in the database is updated with a **"total is known"** flag, indicating:
- The producer has finished enumerating all contacts from the CRM.
- The total expected contact count for the sync is now recorded.

Erik recommended checking the database to verify whether `total is known` was correctly set for the stuck sync before cancelling.

---

## Debugging Session Outcome and Recommendations

- The second full sync (ID `37838`) was still running at the end of the session, stuck due to the audience export timeout.
- **Recommendation:** Cancel the stuck sync and trigger a fresh full sync.
- **Follow-up:** Inspect the database record for the stuck sync to check `total is known` state.
- The profile update that Michal had performed (setting a consent on a topic via form submission) was confirmed as **unrelated** to both issues — the audience service not responding to export was a separate problem from the application-level changes being tested.

---

## Key Takeaways

1. **Full sync has three log locations** — always start by checking whether the sync started (totals visible) before deciding which log group to inspect.
2. **Each ECS task gets its own log stream** — use the full sync ID to find the exact streams; the producer and consumer are always separate streams.
3. **Exit code 2 = "internal system error" / unrecoverable failure** — defined in `FS folder > worker` in the integrations repo. The task-stop handler surfaces this but the messaging can be misleading.
4. **The `concurrent map writes` panic is a race condition** in consumer logic that has been stable for ~2 years — it is anomalous and should be investigated.
5. **Audience export pre-fetch is an optimization, not a hard requirement** — a timeout on this step means slightly redundant consent writes, not data loss.
6. **No callback mechanism exists for consent exports in full sync** — this was a deliberate architectural decision due to complexity of routing async callbacks into ephemeral ECS tasks.
7. **CloudWatch timestamps are UTC** — account for timezone offset when correlating with UI/local times.

---

## Unresolved Questions and Action Items

| # | Question / Action | Owner |
|---|---|---|
| 1 | Root cause of `concurrent map writes` race condition in the consumer — what changed or triggered it? | TBD |
| 2 | Does the producer stop/exit if the consumer crashes (current behavior is unknown)? | Erik to check |
| 3 | Should an Audience export timeout cause the full sync to be marked as failed? | Erik to verify |
| 4 | Inspect database record for stuck sync `37838` — check whether `total is known` flag was set correctly | Michal |
| 5 | Cancel stuck sync `37838` and re-run a fresh full sync on the stage environment | Michal |