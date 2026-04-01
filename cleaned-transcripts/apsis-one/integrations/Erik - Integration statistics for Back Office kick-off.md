---
source_file: Erik - Integration statistics for Back Office kick-off.txt
domain: Apsis One Integrations
topics: [FSC Enterprise installation, webhook conflict on installation, on-premise CRM debugging, full sync producer/consumer termination bug, error handling in defer function, sync job status management]
speakers: ["Erik Andersson (integration engineer/support)", "Tomasz Kowalski (developer)", "Michal Rosikiewicz (developer)"]
key_components: [FSC Enterprise, webhook registration, full sync producer, full sync consumer, ECS tasks, sync job status fields (total_is_known, status), Golang defer function]
session_type: knowledge-transfer
---

## Session Overview

This session covers two distinct bug cases encountered in the Apsis One Integrations domain. The first is a webhook conflict error that blocked FSC Enterprise installation in an on-premise customer environment, and the second is a producer/consumer synchronization bug where a critically failed sync producer does not properly signal termination, causing the consumer to hang for hours before timing out. Erik Andersson walks Tomasz Kowalski and Michal Rosikiewicz through root cause analysis, the preliminary debugging steps applicable across all CRM integrations, and the proposed fix for the sync termination issue. A story/ticket for the fix was confirmed to already exist on the board and will be refined with more detail.

---

## Case 1: FSC Enterprise Installation Failure Due to Pre-existing Webhooks

### Background and Discovery

[Erik Andersson]: This issue was reported by a Professional Services consultant in Benelux/Netherlands. The customer was installing FSC Enterprise 2 (referred to as "FSC Enterprise Dash 2" in the context of migrations) in an **on-premise environment**. On-premise environments frequently cause issues due to IP whitelisting and restricted access.

During the installation steps, when the system attempted to register webhooks, the installation returned a **"page not displayed because there was a conflict"** error. This error is not standard in the general connector — Apsis does not typically receive or handle this type of response.

### Root Cause

After gaining direct access to the customer environment (via a contact who had direct environment and database access), it was discovered that the environment already had **two webhooks pre-configured** — one for consent and one for contacts.

> "FSC Enterprise appears to have added a limitation that you cannot install webhooks if webhooks already exist in the environment."

When those two pre-existing webhooks were deleted, the installation proceeded successfully. Erik was then able to verify the installation by:
- Retrieving metadata
- Retrieving contacts

### How the Pre-existing Webhooks Got There

The origin of the pre-existing webhooks is not definitively known. Two theories:

1. **Failed prior installation attempt** — A previous installation attempt registered the webhooks successfully but then failed at a later step, leaving orphaned webhooks behind.
2. **Manual migration artifact** — If the customer migrated from an older version (e.g., `FC Enterprise` → `FC Enterprise-2`), old webhooks with the previous naming convention may have remained. Erik considers this less likely.

[Michal Rosikiewicz]: Asked whether it's possible the install succeeded at some point and the Apsis One side was deleted but webhooks remained on the CRM side.
[Erik Andersson]: Confirmed this is plausible — a partial or pre-emptive installation could leave webhooks registered without a completed install.

### Key Gotcha for Future Reference

> "If we receive this conflict error from FSC Enterprise during the webhook installation step, the important thing to check is: are there existing webhooks for the environment? If so, they need to be deleted before installation can proceed."

**Caveat on webhook deletion:** Apsis does have the ability to delete webhooks from the environment, but the challenge is knowing the webhook IDs. In this case, the IDs were obtained because the contact could log into the system directly. If that access is not available, the CRM developer team would need to provide the IDs.

---

## Preliminary Debugging Steps for Any CRM Integration (On-Premise or Otherwise)

[Erik Andersson]: These three steps can be applied across any CRM system (Dynamics, Lime, FSC Enterprise 12.0, Tribe, etc.) before escalating to the CRM developer team:

1. **Connectivity check** — Can we reach the environment at all? On-premise environments often have IP whitelisting. Make a manual request and verify there is no timeout or outright permission denial.
2. **Permissions check** — If the environment is reachable, do we have the correct permissions on the key? E.g., can we read contacts or read the schema? Verify we do not receive a `403`.
3. **Data integrity check** — Does the environment return proper structured data, or does it return a large HTML error blob?

> "If we've checked those three things, we've done what we can do. We cannot answer in which circumstances the CRM system gives us a particular error, especially if we haven't seen it before."

**Escalation path:** Issues originating from the CRM system should be forwarded directly to the CRM developer team. Erik noted that currently there are approximately 5 open FSC Enterprise issues that are unanswered, and he is preparing a mail to Ali and Lukasz about this.

---

## Case 2: Full Sync Producer Crash Leaves Consumer Hanging Indefinitely

### Observed Symptom

A sync terminated after several hours, most likely due to a timeout. The sync was never marked as failed. The log output at the end of the run showed a report section very similar to a previously identified timeout issue with consent exports.

### Architecture Context: Producer/Consumer Model

The full sync job consists of two independent components:

- **Producer** — Downloads contacts/data from the CRM system and places messages on a queue. Reports its completion by setting fields in a shared data blob.
- **Consumer** — Independently reads messages from the queue. Terminates when its termination conditions are met.

The shared job state contains (at minimum) two critical fields:

- **`status`** — The current status of the sync job (e.g., running, cancelled, failed)
- **`total_is_known`** — A boolean flag that signals the producer has finished enqueuing all messages

The consumer's termination logic works as follows:
- If `total_is_known` is `true` AND the queue is empty → the consumer knows it is done and exits cleanly.
- If the queue is empty but `total_is_known` is `false` → the consumer assumes the CRM may be experiencing a temporary delay and continues waiting.
- The consumer also checks for `status = cancelled` and presumably `status = failed` (see fix discussion below).
- As a last resort, if nothing arrives for an "obscenely long time" (~several hours), the consumer hits an upper time limit and terminates itself.

### Root Cause of the Bug

When the producer encounters a critical error (in this case, the CRM system returned a large error blob when contacts were being downloaded), the producer shuts itself down by running into its `defer` function, printing a report, and exiting.

**The bug:** The producer's `defer` / cleanup function does **not** set `total_is_known = true` or `status = failed` when it exits due to an error. It only prints the report.

As a result:
- Nothing more is enqueued (producer is dead)
- `total_is_known` remains `false`
- The consumer sees an empty queue but no termination signal, and enters its long wait loop
- Two ECS tasks continue running for ~8 hours unnecessarily

> "The sync is not only the producer — you have an independent consumer task which is dependent on the producer, and that's why we have this big blob of data which in particular contains `status` and `total_is_known`."

### Previously Seen Variant of the Same Bug

[Michal Rosikiewicz]: Noted this matches a previously discussed issue where **consent exports failed repetitively from Audience**, hitting the maximum retry count, which terminated the producer — again without setting `total_is_known` or marking the sync as failed, causing the same consumer hang.

[Erik Andersson]: Confirmed this is the same underlying failure mode but triggered at a different point in the code:
- **Previous case:** Consent export fails, hits max retries, producer exits abnormally
- **Current case:** Contact download from CRM returns an error blob, producer exits abnormally

> "It's a different place in the code, but the outcome will be the same — the producer shuts itself down and `total_is_known` is never updated."

### Proposed Fix

The fix should be applied in the **`everything` function** (the main function of the producer, described as the function where "literally everything happens") — specifically in its **`defer` function** (the Golang `defer` runs after the function exits).

During the cleanup/defer step, the following must be set when the producer exits with an error:

1. **`total_is_known = true`** — so the consumer knows the producer is done
2. **`status = failed`** (or `status = cancelled`) — so the consumer's status-check termination clause is satisfied

[Erik Andersson]: Currently `failed` is only set when something goes wrong in the **consumer**, not the producer. This needs to be symmetric.

Additionally, the consumer's sanity check should be verified/expanded to check for `status = failed` in addition to `status = cancelled`, so that a producer-set `failed` status will cause the consumer to stop immediately.

> "If we then set the status to `failed` in the producer, we would be home safe for both of these cases."

[Michal Rosikiewicz] suggested potentially introducing a new status value like "producer shut down improperly," but Erik indicated the existing `failed` status is sufficient.

### Why This Matters Operationally

- Two ECS tasks run unnecessarily for ~8 hours
- Sync is shown as succeeded in logs even though it failed — customers and support have no visibility into the actual failure
- The underlying CRM error is a separate issue for the CRM team; the Apsis-side fix is purely about clean termination signaling

### Current Status of Fix

A story/ticket for this already exists on the board. Erik will add more detail to it specifying exactly what needs to be set in the `defer` function. Target: Q1 (post-Christmas).

---

## Key Takeaways

1. **Webhook conflict during FSC Enterprise installation** is caused by pre-existing webhooks in the CRM environment. The fix is to delete the conflicting webhooks before re-attempting installation. Webhook IDs must be obtained from someone with direct CRM environment access.

2. **Three universal preliminary debug steps** (connectivity, permissions, data integrity) apply across all CRM integrations and can be performed without CRM team involvement. Anything beyond these requires CRM developer escalation.

3. **The full sync producer does not properly signal failure** to the consumer when it exits due to a critical error. The `defer`/cleanup function in the `everything` function must set both `total_is_known = true` and `status = failed` on error exit.

4. **The consumer's termination logic** relies on `total_is_known` and `status` fields in the shared sync job blob. It currently checks for `cancelled` status but may not check for `failed` — this should be verified and expanded as part of the fix.

5. **This bug has (at least) two known trigger paths:** failed consent export hitting max retries, and a CRM error blob during contact download. The fix in the producer's defer function should address both.

6. **ECS task cost implication:** The bug causes two ECS tasks to run for ~8 hours per occurrence when they should terminate immediately on producer failure.

---

## Unresolved Questions / Action Items

- [ ] **Erik** to add detail to the existing story describing exactly what must be set in the `defer` function (`total_is_known`, `status = failed`)
- [ ] **Developer** to verify whether the consumer's sanity check currently includes `status = failed` as a termination condition, and expand it if not
- [ ] **Erik** to send mail to Ali and Lukasz regarding ~5 unanswered open FSC Enterprise issues
- [ ] **⚠️ Ambiguous:** The exact version naming convention (`FC Enterprise` vs `FC Enterprise-2`, `12.0` vs `12.1`) is used loosely in the transcript. The precise version identifiers should be confirmed against actual system documentation.
- [ ] **⚠️ Unresolved:** The true origin of the pre-existing webhooks in the on-premise FSC Enterprise environment was not definitively determined. Worth confirming with the customer if possible.