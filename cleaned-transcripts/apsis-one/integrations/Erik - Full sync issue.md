---
source_file: Erik - Full sync issue.txt
domain: Apsis One - Integrations
topics: [Full Sync Debugging, Error Code Handling, Log Analysis, ECS Task Management, Consumer-Producer Architecture, Concurrent Map Race Conditions, Audience Export Timeouts, Consent Mapping]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Full Sync Manager, Full Sync Producer, Full Sync Consumer, ECS Tasks, AWS CloudWatch Logs, Audience Service, CRM System, Redis, Consent Export]
session_type: debugging-session
---

## Session Overview

This debugging session focused on troubleshooting a failed full sync operation that produced exit code 2. Erik guided Michal through the multi-layered logging architecture for full syncs, including the Full Sync Manager, Producer, and Consumer components. The team identified a critical "concurrent map writes" race condition in the consumer that caused an unrecoverable panic, and separately discovered an audience export timeout that prevented consent status verification. The session revealed architectural limitations around callback functionality for asynchronous exports.

---

## Full Sync Architecture and Log Location Strategy

### Three-Layer Error Diagnosis

**Full Sync Manager** is the entry point that initiates synchronous requests. If a full sync fails to start immediately after clicking the button, errors appear in the **Full Sync Manager log group**. [Erik Andersson]

The actual sync consists of two separate **ECS tasks**:
- **Producer**: Reads messages from the CRM system and places them into a temporary queue
- **Consumer**: Reads messages from the queue and sends them to the Audience service

Once production and consumption have started (indicated by counting profiles and totals), the issue is not in the Manager logs. [Erik Andersson]

### Navigating CloudWatch Log Groups

When in **stage environment**, there are typically multiple log groups for full syncs. To focus on a specific sync run rather than all events, use the dedicated log streams instead of the "all events" log group. [Erik Andersson]

Each log stream represents one unique full sync, as each sync starts its own task. The stream timestamps indicate when the sync began. When examining multiple syncs:
- Started at 10:19 (first sync)
- Started at 10:21 (second sync)

This allows filtering to the specific execution being debugged. [Erik Andersson, Michal Rosikiewicz]

---

## Race Condition in Consumer: Concurrent Map Writes

### The Fatal Error

During consumer initialization, the process follows this sequence:
1. Setup environments and read configuration
2. Wait for Redis
3. Start worker threads
4. Begin reading messages

At profile 26216, a fatal error occurred: **"Concurrent map writes"** [Michal Rosikiewicz, Erik Andersson]

### Critical Observation

This race condition is deeply problematic because the logic involved **has not changed in 2 years**. [Erik Andersson] The fact that it suddenly manifested suggests either:
- An environmental change or concurrency load increase
- A subtle timing interaction with recent changes elsewhere in the system

> "No idea what's what this would be? I mean like I for for sure it's not something that would occur frequently." [Erik Andersson]

The error is unrecoverable—it triggers a panic in the consumer because the concurrent map writes violate Go's memory safety guarantees. The consumer crashes with exit code 2, reported as "internal system error."

---

## Exit Code 2 and Error Code Mapping

### Code Definition Location

Exit codes are defined in the backend code in the **integration/fs/worker** module. The exit code definitions are referenced but require checking the specific file locations. [Erik Andersson]

### Exit Code 2 Mapping

Exit code 2 corresponds to **"internal system error"** and is surfaced when a task fails. However, there's an important caveat: the task stop service may report this code even when the task didn't fail to start, but rather crashed after starting. [Erik Andersson, Michal Rosikiewicz]

The distinction matters for diagnosis:
- **Failed to start**: Task stop code 1 (task failed to start)
- **Started then crashed**: Should theoretically be different, but exit code 2 masks the actual crash scenario

---

## Producer vs. Consumer Error Propagation

### Typical Error Locations

**Producer errors** (downloading from CRM) are rare unless:
- The CRM has introduced bugs or format changes
- Incorrect API credentials are configured

**Consumer errors** are the typical failure point during processing. [Erik Andersson]

### Task Interdependency

The consumer will exit if the producer fails, because without producer output, a proper sync cannot complete. [Erik Andersson]

> "However, I honestly don't know if the producer stops if the consumer fails." [Erik Andersson]

This asymmetric dependency is an important gap in understanding—the producer may continue running even after the consumer crashes, potentially wasting resources and leaving the sync in an inconsistent state.

---

## Full Sync State Management: "Total is Known"

### Producer Completion Signal

When the producer finishes reading all data from the CRM system, it updates the full sync record in the database with a **"total is known"** flag. [Erik Andersson]

This signal indicates:
- Producer has completed its work
- The total count of contacts to be processed is finalized
- Consumer can now know the expected completion target

Checking this flag in the database helps determine whether the producer successfully completed or crashed mid-operation.

---

## Audience Export Timeout and Consent Mapping

### The Timeout Error

During the failing sync (ID: 37838), the consumer logs showed:

> "Profile export result not ready after 15 minutes" [Michal Rosikiewicz]

This error occurred when the consumer tried to export consent data from the Audience service to compare against incoming CRM data.

### Why Exports Fail: Two Root Causes

**Reason 1**: The Audience service is slow to respond. [Erik Andersson]

**Reason 2** (more insidious): The audience export can fail to trigger entirely, but without callback functionality, the full sync has no way to know. [Erik Andersson]

> "There is an annoying case where like the audience export can fail to trigger, but you don't know that unless you have like the callback functionality" [Erik Andersson]

### Missing Callback Architecture

Full syncs **do not have callback functionality** for consent exports. [Erik Andersson]

**Why?** The architectural challenge is significant:
- Full sync manager starts ECS tasks (producer and consumer)
- Audience service would need to send asynchronous callbacks
- These callbacks would need to route to two separate, ephemeral ECS tasks
- This requires complex networking and state management across service boundaries

At the time of implementation, the team decided this complexity wasn't worth it. Instead, the system uses polling with a 15-minute timeout. [Erik Andersson]

### Consent Export Optimization

During full sync, the system exports consent status for all topics that have a consent mapping configured. The purpose is comparison, not forcing updates.

**Optimization logic**: 
- If a contact is already opted-in in Audience AND we receive opt-in from CRM → skip handling (unnecessary processing)
- If the Audience export times out or fails → **abort the optimization and process anyway**

> "Just to make sure like we we write the opt-in. Even if there were to be an opt in inside of audience like because like this is an optimization part, we don't want to handle the consent if it is already in audience like that, that's an unnecessary processing" [Erik Andersson]

The abort behavior means the sync continues with some extra message processing, which is not functionally incorrect, just less efficient.

---

## Sync Cancellation and Next Steps

### When to Cancel

Michal initiated two full syncs (10:19 and 10:21). The first one crashed due to the concurrent map writes race condition. The second one (37838) was running but had stalled for an hour due to the Audience export timeout.

Decision: **Cancel the running sync and run a fresh one.** [Erik Andersson, Michal Rosikiewicz]

### Database Validation

After cancellation, the team should inspect the database to check the full sync record state, specifically:
- Whether "total is known" flag was set (indicating producer success)
- The sync status and error state
- Profile update counts

---

## Unresolved Questions and Investigation Gaps

1. **Why did the "concurrent map writes" race condition manifest after 2 years of unchanged code?**
   - Was there an increase in concurrency or message volume?
   - Did a dependency update introduce subtle timing changes?
   - Is this profile-specific (profile 26216)?

2. **Does the producer stop if the consumer fails?**
   - Current understanding is asymmetric: consumer stops if producer fails, but the reverse is unclear
   - This affects resource cleanup and sync state consistency

3. **Can the "total is known" flag be reliably checked?**
   - Erik recommended checking the database to understand producer completion state
   - This should be verified after the next sync run

4. **Should full syncs implement callback functionality for Audience exports?**
   - Current 15-minute polling timeout is brittle
   - Callbacks would be architecturally complex but more reliable
   - Tradeoff between reliability and implementation complexity remains unresolved

5. **Are there other profile-specific issues related to recent consent form changes?**
   - Michal wondered if recent changes to consent form submission logic might be related
   - Erik confirmed the current sync doesn't show evidence of this, but broader impact should be investigated

---

## Key Takeaways

- **Full sync debugging requires navigating three distinct log sources**: Manager, Producer, and Consumer, each in separate log groups
- **Exit code 2 is "internal system error"** and typically masks a consumer panic; the crash reason is in the consumer logs, not the exit code
- **Concurrent map writes in the consumer** indicates a race condition that shouldn't occur given 2 years of stable code—investigate environmental or dependency changes
- **Audience export timeouts are tolerable failures**—the system aborts the optimization and processes consents anyway, so a 15-minute timeout shouldn't fail the sync entirely
- **Full syncs lack callback functionality** for async operations, forcing reliance on polling with fixed timeouts; this is an architectural limitation worth revisiting
- **Producer-consumer interdependency is asymmetric**—consumer stops if producer fails, but the reverse is unclear
- **Always check the "total is known" flag** in the database to verify producer completion when diagnosing consumer-side failures

---

## Recommended Actions

1. **Immediate**: Run a fresh full sync after canceling the stalled one (37838)
2. **Short-term**: Check the database state for the failed sync to confirm producer completion status
3. **Investigation**: Determine why the 2-year-stable concurrent map writes logic suddenly failed—check for recent code/dependency changes
4. **Optional**: Investigate whether callback functionality for Audience exports would improve reliability vs. implementation cost