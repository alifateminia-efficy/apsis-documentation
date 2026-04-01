---
source_file: "Erik - Investigate and design solution for approximate age of oldest message.txt"
domain: Apsis One Integrations
topics:
  - PagerDuty alarm design philosophy and limitations
  - Outbound worker queue management and FIFO queue clogging risk
  - SQS retry/backoff logic refactoring
  - Mock CRM service for local/staging testing
  - Emergency queue drain procedure
  - CPU alarms for Delta Sync and Sluice workers
  - Monitoring dashboard design discussion
speakers:
  - "Erik Andersson (Senior/Platform Engineer, knowledge holder)"
  - "Michal Rosikiewicz (Team member, on-call)"
  - "Tomasz Kowalski (Team member)"
key_components:
  - Outbound Worker
  - Sluice Worker
  - Delta Sync Worker / Delta Sync Manager
  - SQS (AWS Simple Queue Service)
  - Mock Service / Echo Server (generic connector mock)
  - PagerDuty
  - CloudWatch / Log Insights
  - Manager Cloud Formation
  - SAM file (sluice worker)
  - lib/sqs (Go package, selectTimeout function)
session_type: knowledge-transfer
---

# Session Overview

This session covers the design philosophy and known limitations of PagerDuty alarms in the Integrations domain, with a focus on the outbound worker's SQS queue and the problems caused by continuously failing messages that never auto-resolve. Erik walks through a proposed refactor of the exponential backoff/retry logic in the SQS library, explains how to use the internal mock CRM service to test the new behavior in staging, and demonstrates an emergency procedure for draining a clogged outbound queue. The session also briefly covers CPU alarm tuning for the Delta Sync Manager and Sluice Worker. A monitoring dashboard concept is raised but deferred pending Lukasz's return.

---

## PagerDuty Alarm Design Philosophy and the Auto-Resolve Problem

### How Alarms Are Supposed to Work

The intended behavior for PagerDuty alarms is:
- An alarm triggers on an actual error.
- If no further errors occur within a defined window, the alarm **auto-resolves**.
- If the same issue recurs after resolution, a new alert fires — this is the signal that action is still required.

This model primarily applies to **manager services** (customer integration setup, MAC triggers, full syncs). Errors in those services typically originate from:
- Things going wrong inside apps
- Customers doing unexpected things (e.g., inputting malformed data)

### The Problem: Continuous Errors in the Outbound Worker

[Erik Andersson]: For the **outbound worker**, errors are almost always connected to the CRM system — a CRM returns an error when we POST to it. We cannot fix bugs in the CRM system directly. The only available action is to notify the relevant party (SOC for permission issues, CRM developers for unexpected errors/bugs).

The critical issue: **some customers have continuously failing messages, so the alarm never auto-resolves**. It just stays in alert state indefinitely. You can have an unlimited number of errors accumulating and the team has no visibility into the growing volume, because the alarm never transitions out of the alert state.

> "It just triggers and then there can be as many errors as humanly possible, but we don't know that because it is still in the alert stage because it has not yet auto-resolved."

### Why the Outbound Worker Queue Cannot Be Ignored

The outbound queue is a **FIFO SQS queue**. If the queue reaches the **20,000 message limit** for a given message group and stays there too long, it causes a serious problem:

> "ABS [the queue processor] only looks for message groups in the first 20,000 messages. If those 20,000 messages are laying there being timed out, that will clog up the whole outbound worker queue."

Example scenario: A customer is blocking email-sending events. They accumulate 20,000 events of the same type. Any message from any other customer that lands behind those 20,000 messages will not be processed — because the processor never sees past the first 20,000. This can stall processing for all customers.

**Conclusion:** The team cannot simply convert outbound worker errors to warnings and ignore them. Some level of visibility and actionability is required.

---

## Monitoring and Dashboard Design (Deferred)

### Current On-Call Process

[Michal Rosikiewicz]: Currently the team does weekly post-on-call log reviews using **CloudWatch Log Insights** queries. Based on results, they either plan an internal fix or notify the customer that something is failing on their end.

### Proposed Dashboard Approach

[Erik Andersson]: A more proactive approach would be to collect failed message statistics into a **metrics cache or database entry**, and surface these on an **AWS CloudWatch dashboard** per account/section/integration. This would give an at-a-glance view of overnight failures.

The dashboard could potentially aggregate all actionable error types:
- CRM POST failures
- Permission errors (e.g., 401 from rotated API keys)
- Data retrieval errors

[Erik Andersson]: Some of these (permission errors, rotated API keys) could even be routed directly to SOC automatically — but that would require a redesign of how Integration currently handles these errors.

> ⚠️ **This redesign was not fully designed in this session.** Both speakers agreed it is a "process question, not just a technical implementation question." Decision: defer full monitoring redesign until Lukasz is available. The backoff refactor (below) is a good first step.

---

## SQS Retry/Backoff Logic Refactor

### Current State (and Why It's Bad)

The current retry logic uses an exponential backoff with approximately **8 maximum retry attempts**, with delays escalating roughly as:
```
15s → 15s → 1min → 5min → 15min → 1hr → 4hr → 6hr (→ ~5hr)
```
Total retry window: approximately **1.5 days**.

The current implementation does **not** use the native SQS message metadata field `ApproximateReceiveCount` (how many times a message has been received). Instead, it calculates backoff time based on elapsed wall-clock time and compares against thresholds — a fragile and convoluted approach.

[Erik Andersson]:
> "This function should be obliterated from orbit because it is so ugly."

The relevant function is:
```
lib/sqs/select_timeout.go  (selectTimeout)
```

### Problems with the Current Retry Window

[Michal Rosikiewicz]: If a CRM has legitimate downtime (e.g., 20 minutes), the current 4–6 hour intervals mean messages will eventually get through. If we reduce intervals to 15 minutes, we need more retries to cover a similar total window.

[Erik Andersson]: The 1.5-day total retry window is overkill.

**Agreed direction:**
- Reduce retry interval to **15 minutes** (flat after initial shorter steps)
- Set a **maximum total retry window of ~5 hours**
- Calculate number of retries as: `5 hours / 15 minutes = 20 retries` (approximately; final number to be agreed — Erik suggested ~75 in a later pass when accounting for the stepped early intervals, but flagged this needs validation)

> ⚠️ **Ambiguity:** The exact final retry count was not firmly agreed. Erik worked through the math live and landed on "~75" as a rough ceiling, but this should be validated during implementation.

### Proposed New Implementation

Replace the current backoff logic with a switch-case on `message.ApproximateReceiveCount` (the native SQS metadata field):

```go
// In lib/sqs/select_timeout.go
// Function signature needs to accept the SQS message (sqs.Message) to access metadata

switch amountOfRetries { // derived from message.ApproximateReceiveCount
case 0:
    return 5 * time.Second
case 1:
    return 30 * time.Second
case 2:
    return 1 * time.Minute
case 3:
    return 5 * time.Minute
default:
    return 15 * time.Minute
}
```

Where `selectTimeout` is called, the caller is already iterating over SQS messages, so it has direct access to the message struct and its metadata.

The existing **maximum retry check** (which discards messages after the limit is reached) must be preserved — just updated to use the new retry count and ceiling.

[Erik Andersson]:
> "Compared to how it used to be — it's so much more prettier."

### Where This Logic Is Used (Scope of Change)

The backoff logic needs to be verified/updated in **two places**:
1. **Sluice Worker** — backoffs applied when a full sync is running
2. **Outbound Worker** — backoffs applied when CRM system errors occur

It does **not** apply to the full sync worker directly.

---

## Testing the Backoff Refactor Using the Mock CRM Service

### What the Mock Service Is

[Erik Andersson]: There is an internal **mock CRM service** (a "generic connector" implementation) used for local and staging development. It fulfills the generic connector contract, allowing you to simulate any supported CRM system without needing access to a real CRM.

The mock service is configured via the **API key field** when setting up an installation — the API key value is the **CRM logical ID** you want it to impersonate:

| API Key Value | CRM Simulated |
|---|---|
| `FSCCorporate` (or similar) | FSC Corporate |
| `FECU` | Maxo/Maxwell |
| *(etc.)* | *(other supported CRMs)* |

**How to identify a mock installation:** All field attributes will be prefixed with the entity name (e.g., `person_email`, `person_firstname`) rather than real CRM field names.

The mock service supports:
- Field mappings
- Sync conditions
- Consent messages
- Email/SMS/form/event tool events → responds with a random UID as a fake CRM identifier ("Trust me bro")
- Full syncs → returns predictable, incrementing data (e.g., `firstname_1`, `firstname_2`, ...) up to ~10,000 records, not random data, so sync conditions and segments are testable

**Known limitation of the mock service:** It does **not** currently support real-time messages *from* the mock service *to* Apsis. This is technically implementable (trigger an API call on demand) but has not been built.

### How to Set Up a Mock Installation on Staging

Navigate to the staging environment integration setup and enter:
- **Address:** URL of the mock service
- **API key:** The logical CRM ID you want to simulate (e.g., `FSCCorporate`)

### How to Test the Backoff Logic

To simulate CRM errors for backoff testing, modify the mock service's post-campaign events handler:

```go
// In mock service: echo_server.go (or server.go)
// In the post campaign events function:

if body.Account == "<your_account>" && body.Section == <your_section> {
    // return HTTP 500 Internal Server Error
}
```

This scopes the error to your specific account/section only, avoiding disruption to other developers using the mock service simultaneously.

**Deployment note:** This mock service change should be **deployed manually to staging** (outside of your feature branch / PR) — push the change directly, do not include it in the pull request for the backoff refactor itself.

[Erik Andersson]: Estimated effort for backoff refactor + testing: **2 story points**.

---

## Emergency Queue Drain Procedure (Clogged Outbound Queue)

### When This Is Needed

If a customer sends a massive batch (e.g., 500,000 events in one go) and the outbound worker gets stuck processing them, blocking all other customers from having their messages processed — potentially for days — a manual queue drain is required.

> "It's better that those events are not sent rather than us not sending anything for any customer for potentially days."

### How to Do It

In the **Outbound Worker**, locate the **message processor** (`message_processor.go` or similar). Inside the `processMessage` function, the **integration key (IK)** is available, which contains `account`, `section`, and `integration` identifiers.

To silently drop all messages for a specific problematic integration, add a guard at the top of `processMessage`:

```go
if ik == problematicIntegration &&
   ik.Account == accountThatCausesIssues &&
   ik.Section == sectionThatCausesIssues {
    logger.WarnD("skipping message due to clogged queue", /* log fields */)
    return nil
}
```

**Why `return nil` works:** Returning `nil` (no error) signals that the message was successfully processed. The SQS framework will then delete the message from the queue. No CRM call is made. This drains tens of thousands of messages within approximately **one minute**.

### Important Caveats

> ⚠️ **This constitutes data loss.** Do not do this without consulting the team and getting explicit approval.

Alternative to pure discard: move messages to the **dead letter queue (DLQ)** and redrive them later, depending on the situation.

**Limitation:** This only works for messages that are currently **visible** in the queue. If messages are in a long backoff window (invisible), you must wait for them to become visible before they can be read and dropped. This is another argument for reducing the maximum backoff window to 15 minutes — it means you can drain a stuck queue much faster.

> "There are literally only upsides for [the 15-minute max backoff]." — Erik Andersson

---

## CPU Alarms: Delta Sync Worker and Sluice Worker

### The Problem

[Michal Rosikiewicz]: High CPU alerts on the Delta Sync Worker were firing multiple times during this meeting session. This is a recurring false-positive-style alarm pattern.

### Root Cause Context

[Erik Andersson]: The Delta Sync Worker and Sluice Worker are tightly coupled in behavior:
1. If a customer changes their CRM metadata structure, **every contact** in their CRM gets flagged as updated.
2. The **Delta Sync Manager** is then bombarded with this massive sync load.
3. Those contacts are moved from the Delta Sync Worker to the **Sluice Worker** — so a CPU spike in Delta Sync almost always precedes a spike in the Sluice Worker.

### Alarm Configuration Locations

- **Sluice Worker CPU alarm:** Contained within the Sluice Worker's own **SAM file** — can be changed independently.
- **Delta Sync Manager CPU alarm:** Contained in the **central Manager CloudFormation template** — any change there affects all managers. Erik's assessment: a more lenient CPU alarm is appropriate for all managers, so changing it centrally is acceptable.

### Proposed Fix

[Michal Rosikiewicz]: Standard practice on the team is to use **5 consecutive evaluation periods of ~5 minutes each** = alarm only fires if CPU is elevated for **25 consecutive minutes**. Currently the integration services appear to use shorter periods, causing false alerts on legitimate burst traffic.

[Tomasz Kowalski]: Need to examine how the current alarms are parameterized before adjusting.

**Agreed approach:** Compare against historical CloudWatch graphs. If a past alarm was not a "real" problem, tweak the evaluation period/threshold so it would not have triggered. The same treatment applied to the Sluice Worker backoff should be applied to the Delta Sync Worker alarm.

---

## Key Takeaways

1. **The outbound worker alarm never auto-resolves when errors are continuous** — the team effectively has no visibility into how many errors are accumulating. This needs to be addressed either via a monitoring dashboard or a redesign of the alerting model.

2. **The FIFO queue 20,000-message limit is a real operational risk.** If a single customer/message-group fills the first 20,000 slots, all other customers are blocked. Active monitoring and the ability to drain the queue quickly are essential.

3. **The current `selectTimeout` function in `lib/sqs/select_timeout.go` is the single point to change for retry behavior** — refactoring it to use `ApproximateReceiveCount` (native SQS metadata) instead of wall-clock-time heuristics will make it dramatically cleaner and more reliable.

4. **Reduce maximum retry interval from the current multi-hour exponential steps to 15-minute flat retries with a ~5-hour total window.** This reduces the time a message is invisible/locked, enabling faster queue drains if needed.

5. **The mock CRM service is a powerful, underutilized testing tool.** It can simulate any supported CRM, return predictable data for sync condition testing, and be configured to return errors for specific accounts — all without touching real CRM systems. It is set up via the API key field using the CRM's logical ID.

6. **The emergency queue drain (`return nil` in `processMessage`)** is a valid but consequential tool. Always get approval before using it. Consider the DLQ redrive path if data preservation is important.

7. **The Delta Sync Worker and Sluice Worker CPU alarms need retuning** — burst traffic is expected behavior; the alarm should only fire on sustained elevated CPU (e.g., 5 consecutive 5-minute periods).

---

## Unresolved Questions and Action Items

| Item | Owner | Notes |
|---|---|---|
| Agree on final retry count and interval schedule (e.g., 75 retries? 5-hour window?) | Michal / Tomasz / Erik | Erik's live math was approximate; needs validation |
| Implement `selectTimeout` refactor using `ApproximateReceiveCount` | Tomasz (likely) | Estimated 2 points; verify in Sluice Worker and Outbound Worker |
| Add mock service 500 error for specific account/section to test backoff | Team | Deploy manually to staging, outside of PR |
| Design full monitoring/dashboard solution | Deferred | Wait for Lukasz; dashboard in AWS CloudWatch is the proposed direction |
| Determine whether to remove or radically change the PagerDuty alarm if a dashboard replaces it | Erik / Team | Risk of alarm staying in permanent alert state if dashboard approach is adopted |
| Tune Delta Sync Manager CPU alarm in central Manager CloudFormation | Team | Review historical data first; likely applies to all managers |
| Tune Sluice Worker CPU alarm in SAM file | Team | Likely 5 × 5-minute evaluation periods |
| Implement real-time message triggering in the mock service (from mock → Apsis) | Backlog | Not urgent; noted as a known gap |