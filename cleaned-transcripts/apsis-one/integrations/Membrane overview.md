---
source_file: Membrane overview.txt
domain: Apsis One Integrations
topics: [outbound queue architecture, FIFO queue blocking, CRM error handling, activity grouping fix, DevOps monitoring gaps]
speakers: ["Erik Andersson (Engineer/Developer)", "Lukasz Grabowski (likely Product Owner or Engineering Lead)"]
key_components: [outbound queue system, CRM integration, form submission pipeline, activity type sub-queues, APSIS integrations]
session_type: knowledge-transfer
---

## Session Overview

This session covers a production incident where an Ed-layer customer experienced delays of over one day on form submissions reaching the CRM system. Erik explains the root cause in the outbound queue architecture — specifically how FIFO queuing grouped by activity type causes one failing activity to block all other activities of the same type. He proposes a minimal fix (adding activity ID to the grouping key) and also raises a broader concern about the current state of DevOps ownership across the integration. The session ends with a plan to add the fix to the upcoming sprint.

---

## Incident Description: Form Submission Delays to CRM

A customer (Ed / E-deal) experienced delays of over a day — approximately 20 hours — for form submissions to arrive in the CRM system.

The **root cause** was two-fold:
1. The CRM system was responding with errors when APSIS submitted forms.
2. The APSIS outbound queue architecture amplified this into a prolonged blockage.

### Specific Failure Scenario

- The customer had set up an attribute mapping in the outbound configuration. When APSIS sent that attribute to E-deal, E-deal attempted to update with it and returned a **database error** on every form submission.
- This is described as a **permanent error** on the CRM side, but because it arrives as an unexpected/generic error (not a recognized permanent failure code), APSIS cannot distinguish it from a transient error.

> "The errors they are giving us are permanent errors, but we don't know that because it comes in the form of an unexpected error. So should APSIS retry this or should we discard this? We don't know. And if we don't know, the only safe way is to do a retry."

- The customer had 5–6 failing form submissions in the queue, each being retried for up to **4 hours** (per the retry/backoff policy introduced by Thomas, with backoff up to 15 minutes and a total retry window of 4 hours).
- The customer then created a **new test form**, which was placed at the back of the queue behind the already-failing messages.
- Result: ~20 hours of delay before the new test form submission was processed.

---

## Outbound Queue Architecture and the FIFO Blocking Problem

### Queue Structure

The outbound flow is organized as follows:
- One **internal queue per customer account** and integration.
- Within each customer's queue, there are **sub-queues per activity type**: e-mail events, SMS, forms, and every other APSIS activity type.
- These sub-queues are **FIFO** (First In, First Out).

### How Blocking Occurs

Because all form submissions across all form activities go into a single FIFO sub-queue (grouped only by account + activity type), a failure on **Form A** blocks **Forms B and C** from being processed — even if Forms B and C would succeed.

The policy rationale behind this design was:

> "If they fail to process one form, it is a high likelihood they will fail to process any other form."

While this reasoning has some validity, the practical effect is that one misconfigured or problematic form activity can hold up all other form submissions for the full retry window (up to 4 hours per message, multiplied across multiple failing messages).

**Current grouping key:** `[account] + [integration] + [activity type]`

---

## Proposed Fix: Add Activity ID to Queue Grouping Key

### The Change

Erik proposes adding the **activity ID** to the grouping key used for queue routing:

**New grouping key:** `[account] + [integration] + [activity type] + [activity ID]`

This means:
- Each individual form activity gets its own sub-queue.
- A backoff/retry on Form A does **not** block Form B or Form C.
- Activities that are succeeding can be processed in parallel with those that are failing.

### Implementation Scope

> "This is a trivially small change for us. This is like a one-liner change."

Erik was aiming to implement this on Monday (the start of the third sprint). No story existed yet at the time of the session — Erik identified this fix the previous evening (9:30 PM) while reviewing a potential incident.

### Limitations of the Fix

This fix reduces the blast radius of CRM errors but does **not** eliminate delays caused by the CRM returning errors. If a form activity is consistently receiving errors from the CRM, submissions for that specific activity will still be delayed up to 4 hours total. The fix only prevents that failure from spilling over into other form activities.

---

## DevOps and Monitoring Gap

Erik raised a significant process concern beyond the technical fix.

### Current State (Problematic)

- All DevOps work for the integration is effectively owned by APSIS R&D.
- APSIS R&D has been sending form submissions to the CRM and **causing database errors in the CRM system** — without the CRM team being aware until Erik proactively notified them.
- The flow of information is consistently one-directional: APSIS tells CRM teams about errors in CRM's own system, never the other way around.

> "It is never that anyone comes to us and says hello, we are experiencing errors from this customer. It is always APSIS that comes to the CRM teams and says hello, we are experiencing errors in your system. And this really must change going forward."

### Impact

This specific incident could have been resolved in **2–3 hours** if the CRM team had monitoring in place. Instead it went on for days.

### Erik's Position

Erik is not suggesting APSIS should stop proactive notification, but that **CRM teams must also have their own monitoring** and must inform APSIS when errors are occurring on their end. The current state is not an acceptable way of working.

---

## Sprint Planning and Action Items

- **Fix (add activity ID to grouping key):** High priority, to be included in Sprint 3 starting Monday. No story exists yet — needs to be created.
- **Assignee consideration:** Check whether Tomac (currently finishing work on MA/a script) can be involved.
- **Email:** Erik to send the full detailed email (written the previous evening explaining the queue issue and proposed fix) to Lukasz.
- **Recipient of email:** Primarily addressed to Ed (the affected customer), given they are experiencing the most significant form submission issues.

---

## Key Takeaways

1. **The FIFO sub-queue-per-activity-type design causes cross-activity blocking.** One failing form submission holds up all other form submissions for that customer for the entire retry window (up to 4 hours, potentially compounded across multiple failing messages).
2. **Ambiguous error responses from CRMs force conservative retry behavior.** When errors cannot be classified as permanent vs. transient, APSIS defaults to retry — which is the safe choice but creates long delays when errors are actually permanent.
3. **A one-liner fix — adding activity ID to the grouping key — significantly reduces blast radius** by isolating backoffs to individual activities rather than all activities of a type.
4. **The current retry policy (Thomas's change): backoff up to 15 minutes, total retry window of 4 hours.**
5. **DevOps ownership is currently asymmetric and unsustainable.** CRM teams need their own monitoring and alerting for integration errors; APSIS R&D should not be the sole party detecting errors in third-party systems.
6. **Fragile outbound attribute mapping** is a contributing factor — a single misconfigured attribute in the APSIS-to-CRM outbound mapping can cause the CRM to return errors on every subsequent submission, killing the entire flow for that activity.

---

## Unresolved Questions / Action Items

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Create story for activity-ID grouping key fix | Lukasz / Erik | To do — include in Sprint 3 |
| 2 | Erik to send detailed email to Lukasz | Erik | Verbally confirmed |
| 3 | Check Tomac's pipeline availability for the fix | Lukasz | Pending |
| 4 | CRM teams to establish their own monitoring/alerting | Process/stakeholder concern — no owner assigned | Unresolved |
| 5 | How to properly classify permanent vs. transient errors from CRM to avoid unnecessary retries | Not discussed in this session | Unresolved |