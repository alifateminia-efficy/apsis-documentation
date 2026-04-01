---
source_file: Erik from Nov 27th 2025.txt
domain: Apsis One Integrations
topics: [full sync debugging, ECS task architecture, log navigation, exit codes, race conditions, audience export timeouts, consent mapping]
speakers: ["Erik Andersson (senior engineer / domain expert)", "Michal Rosikiewicz (engineer, screen-sharing)", "Tomasz Kowalski (engineer, observer)"]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS tasks, CloudWatch Log Groups, Audience service, Redis, CRM integration]
session_type: debugging-session
---

# Full Sync Debugging Session — November 27, 2025

## Session Overview

This session was a live debugging walkthrough led by Erik Andersson, with Michal Rosikiewicz sharing their screen. The team was investigating a failed full sync in the staging environment, which had produced exit code 2 and a `concurrent map writes` fatal error. The session covers how to navigate CloudWatch log groups for full sync issues, the producer/consumer ECS task architecture, how exit codes are set in the codebase, an audience export timeout encountered during the session, and the architectural rationale for why callback functionality was not implemented in full syncs.

---

## Full Sync Architecture: Producer and Consumer ECS Tasks

A **full sync** consists of two separate ECS tasks that run concurrently:

- **Full Sync Producer**: Reads contact/profile messages from the CRM system and places them into a temporary queue.
- **Full Sync Consumer**: Reads messages from that temporary queue and sends them into the Audience service.

The **Full Sync Manager** is a separate component that handles the synchronous request to *start* a full sync. It is only relevant if the sync fails to start at all (i.e., you click "full sync" and it immediately fails before doing anything).

> "The manager is like this synchronous requests. The full sync manager is the one that starts a full sync. If a full sync fails to start, like you click on full sync and it immediately fails, then it would be in the full sync manager log group you should check."

**Key indicator that the manager is NOT the problem**: If the sync has already begun counting/totalling profiles, the producer and consumer have started, and the manager is no longer in scope for debugging.

### Consumer/Producer Dependency Behavior

- If the **producer fails**, the consumer will exit — because without a producing side, a proper sync cannot complete.
- Whether the **producer stops if the consumer fails** was noted as unclear.

> "I honestly don't know if the producer stops if the consumer fails."

⚠️ **Unresolved**: The exact shutdown behavior when the consumer fails (not the producer) needs to be verified in the code.

---

## Navigating CloudWatch Log Groups for Full Sync Debugging

### Log Group Structure

There are (at minimum) two relevant log groups for full syncs:

1. **Full Sync Producer** log group
2. **Full Sync Consumer** log group

Each individual full sync run creates its own **log stream** within those groups. Each log stream represents one unique full sync task execution.

### How to Find the Right Log Stream

- Do **not** use the "all events" view when investigating a specific sync — in staging, multiple syncs may be running and their events will be interleaved.
- Navigate to the dedicated log group (producer or consumer), then identify the correct log stream by:
  1. Checking the **start timestamp** of the log stream.
  2. Or searching by the **full sync ID** (visible in the UI on the sync's detail page, e.g., `37838`).

> "If you want to check on one specific sync, it's better to check in the dedicated log streams instead."

### Reading the Consumer Log Stream

The top of a consumer log stream will show initialization:
- Setting up environments
- Reading configuration
- Waiting for Redis
- Starting worker threads
- Beginning to read messages

The session identified the following in the consumer log for the failing sync:

- **`concurrent map writes`** — fatal error / panic indicating a race condition
- This panic caused an abrupt, unrecoverable exit

---

## Exit Code 2: Where It Is Set and What It Means

**Exit code 2** is surfaced in the backend. The session traced the location:

- The exit codes are defined in the **`FS` folder** (full sync folder) within the `integration` repo — specifically in the **`worker`** file (not the Full Sync Manager subfolder, but the `FS` folder above it).
- The relevant exit code is associated with **"unrecoverable failure"** (the comment in the code describes it as such).

However, the session found that the exit code 2 is actually being reported via the **Task Stop** mechanism in ECS. This was identified as potentially misleading:

> "I think the task stop is giving the incorrect code for it, because the task has not failed to start — it has started. It's just that it is crashing horribly when it started."

The specific error path observed:
- "Internal system error" → exit code 2
- This maps to the task-stopped handler (path searched was something like `AskStop` / task stopped events), not a clean startup failure.

---

## `concurrent map writes` Fatal Error (Race Condition)

The consumer log for full sync `37838` (started ~10:21) showed a **fatal `concurrent map writes` panic** at profile ID `26216`.

Erik's assessment:

> "This concurrent map writes — there seems to be some race condition going on, which is weird because this logic has not changed for like 2 years."

- Erik confirmed he had no immediate explanation for it.
- It is not expected to occur frequently.
- The profile in question (`26216`) was not necessarily touched by the current team — in a full sync, all profiles in the CRM are downloaded, not just recently modified ones.

⚠️ **Unresolved / Action Item**: The race condition needs further investigation. It may be a rare, intermittent issue given the code's age and stability.

---

## Audience Export Timeout During Full Sync

A separate issue was found in the **producer** log stream for the second sync (full sync ID `37838`, log stream `F3C4...`):

```
Profile export result not ready after 15 minutes
```

This is an **audience export timeout**. Erik explained the mechanism:

### How Consent Export Works During Full Sync

When running a full sync, the producer triggers **exports of existing consent statuses from the Audience service** for all topics that have a consent mapping configured. This is done to compare incoming CRM consent data against what is already stored in Audience:

- If a contact already has an **opt-in** in Audience for a topic, and the CRM is also sending an **opt-in** for that topic → the full sync **skips** processing that consent entry (optimization: no unnecessary writes).
- If the audience export **fails or times out**, the full sync producer logs an "aborting" message for that consent check.

> "What I think it means is we just don't check that consent status in Audience. So if we get an opt-in for the contact for that specific topic, we still handle that opt-in... we write the opt-in even if there were to be an opt-in inside of Audience already — because this is an optimization part, we don't want to handle the consent if it is already in Audience."

**Consequence of timeout**: The sync may process more messages than strictly necessary (redundant opt-in writes), but it is **not inherently a fatal or incorrect outcome**. Erik indicated he does not believe this alone should cause the sync to fail.

> "I honestly don't know if this sync should have failed at that stage or not. I don't think it should have, in all honesty."

⚠️ **Unresolved**: Whether the audience export timeout causes the full sync to be marked as failed, or just results in extra processing, needs to be verified.

---

## Why Full Sync Does Not Have Callback Functionality for Consent Exports

This is a deliberate architectural decision made at build time:

> "The full syncs don't have any endpoints to receive callbacks to — and at least at the time of building we opted to not have that, because you would trigger something from the full sync manager which will start ECS tasks which will then need to get asynchronous callbacks from some other service, from Audience, which then needs to go to these two separate ECS tasks. It was not a funny thing."

The complexity of routing async callbacks to ephemeral ECS tasks was deemed not worth it. The tradeoff is that the full sync uses polling with a timeout (15 minutes) instead of a callback.

---

## Database State Check: `total_is_known` Flag

Erik mentioned that after the producing phase completes, the full sync record in the **database** is updated with a flag indicating that the total number of contacts is now known:

> "What we do in the full sync producer is that when the producing part is done, we update the full sync in the database with 'total is known', meaning the producer has finished — we know how many contacts should be in the sync."

**Recommended action** (from Erik): Before or after cancelling a stuck sync, check the database record for the full sync to verify whether `total_is_known` was set, which would confirm the producer completed successfully.

---

## Recommended Debugging Steps for a Failing Full Sync (Summary)

1. **Did it fail immediately on start?** → Check **Full Sync Manager** log group.
2. **Did it start (profiles counted/totals shown)?** → Check **Producer** and **Consumer** log groups.
3. Navigate to the specific **log stream** (not "all events") using the sync's timestamp or ID from the UI.
4. Look for fatal errors / panics at the **top** of the consumer log stream.
5. Look for timeout errors (e.g., `Profile export result not ready after 15 minutes`) in the **producer** log stream.
6. Exit code 2 = "internal system error / unrecoverable failure" — surfaced via ECS Task Stop handler; check `FS/worker` in the `integration` repo for exit code definitions.
7. Verify the **database record** for the `total_is_known` flag to assess producer completion state.
8. If the sync is stuck/failed → **cancel and re-run**.

---

## Key Takeaways

- **Three log locations** matter for full sync issues: Full Sync Manager (start failures), Producer log group, Consumer log group — always use dedicated log streams, not "all events."
- **Producer and Consumer are separate ECS tasks** — they have different log streams for the same sync ID. When comparing logs, be explicit about which one you are looking at.
- **Exit code 2** = unrecoverable failure; defined in `integration` repo under `FS/worker`; surfaced somewhat misleadingly through the ECS Task Stop handler.
- **`concurrent map writes` panic** is a rare, unexpected race condition in logic that has been stable for ~2 years — warrants investigation but likely not caused by recent changes.
- **Audience export timeout** during full sync causes extra (redundant) consent processing but is likely not fatal to the sync itself — the optimization is skipped, not the sync.
- The **absence of callback support** in full syncs is intentional — the architectural complexity of routing callbacks to ephemeral dual ECS tasks was rejected at build time.
- The **`total_is_known` flag** in the database is a useful diagnostic to confirm whether the producer phase completed successfully.

---

## Unresolved Questions and Action Items

| # | Question / Action | Owner |
|---|---|---|
| 1 | Investigate the `concurrent map writes` race condition in the full sync consumer — what changed or what conditions triggered it? | TBD |
| 2 | Confirm: does the producer stop/exit if the consumer fails (not the other way around)? | Erik to check |
| 3 | Confirm: does an audience export timeout cause the full sync to be marked as failed, or just result in extra processing? | Erik to check |
| 4 | Check the database record for full sync `37838` — was `total_is_known` set (i.e., did the producer complete)? | Michal |
| 5 | Cancel sync `37838` and re-run a fresh full sync | Michal |