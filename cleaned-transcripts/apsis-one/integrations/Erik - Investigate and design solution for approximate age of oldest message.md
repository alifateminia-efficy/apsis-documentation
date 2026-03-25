---
source_file: Erik - Investigate and design solution for approximate age of oldest message.txt
domain: Apsis One Integrations
topics: [Outbound Worker Architecture, Message Queue Management, Retry Logic and Backoff Strategy, Error Handling and Monitoring, Alert Configuration, Queue Clogging Prevention, Mock Service Testing]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Outbound Worker, SQS Queue, Delta Sync Worker, Sluice Worker, Mock Service, CRM Systems, Message Processor]
session_type: architecture-review
subdomains: [Architecture, Outbound Flow, Generic Connector]
---

## Session Overview

This session focused on redesigning the message retry and backoff logic in the outbound worker to address ongoing issues with alert management, queue clogging, and system visibility. The team discussed the current exponential backoff implementation (which spans approximately 1.5 days), the limitations of the existing alarm system that doesn't properly handle continuous errors, and designed a cleaner, more manageable retry strategy. Key decisions included reducing total retry time to approximately 5 hours, improving the backoff function to use SQS message metadata directly, establishing monitoring dashboards for failed messages, and setting up testing procedures using a mock CRM service.

---

## Page Duty Alarm Design Philosophy and Current Problems

### Alert Trigger and Auto-Resolution Mechanism

The current design philosophy for page duty alarms is as follows:

> "Whenever there is an actual error in the systems, the alarm will trigger and if there has not been any more errors for a given time then it will auto resolve. If the same issue occurs again, hopefully you would have actually done something about it. Otherwise you should get pinged until some kind of action is taken upon it."

[Erik Andersson] This approach works well for **manager services** where there's actual customer integration—when customers trigger full syncs or input malformed data, errors are discrete and recoverable.

### Limitation with Continuous Errors

The critical problem arises with **continuous, unresolved errors**, particularly in the **outbound worker**:

[Erik Andersson]: "The problem here now is that there are continuously errors for some customers. So this like never auto resolves itself. It just triggers and then there can be as many errors as humanly possible, but we don't know that because it is still in the alert stage because it has not yet auto resolved, which might not be the most optimal situation."

The alarm stays in "alert" state perpetually, providing no visibility into the volume of failures or their frequency.

### Why Outbound Worker Errors Are Different

Outbound worker errors are almost always connected to **CRM system issues**, and the integration team cannot directly fix them:

[Erik Andersson]: "If we post something to the CRM system and they respond with an error and we register this as an error in return, we can't go in and do bug fixes in the CRM system. We need them to take action on that. What we can do there is to notify whoever we need to notify that this happened, we are failing to provide you with the data we are supposed to provide to you for events."

Depending on error type:
- **Permission issues** → notify SOC (System Operations Center)
- **Unexpected bugs/errors** → notify CRM developers
- **Consent failures** → more critical than other failures

---

## The Queue Clogging Problem: Why Visibility Matters

### FIFO Queue and Message Group Processing Limits

The outbound queue is a **FIFO queue** with a critical constraint. [Erik Andersson] explained the risk:

> "If we cross this 20,000 messages in the queue limit for too long of a time, we might risk ending up in some nasty situation because of how ABS looks for the message groups to process. If you might remember, let's say that I have 20,000 messages in the queue for the same customer for the same type. For example, they are blocking an email sending events from us and we have 20,000 events from the same customer. Then any message that is behind those 20,000 messages will not be processed because ABS only looks for message group in these first 20,000 messages and they are laying there being timed out and that will clog up the whole outbound worker queue if left unmaintained."

### Consequence of Inaction

- Messages from all other customers back up and stop processing
- Critical operations (consents, events, campaigns) stall across the platform
- The integration team cannot simply ignore the problem by downgrading errors to warnings

---

## Proposed Solutions for Monitoring and Alerting

### Option 1: Dashboard-Based Monitoring System

[Erik Andersson] proposed a statistics/metrics-based approach:

> "If you'd develop some kind of a system here where if we fail to process any message in the outbound worker, we add this to some statistic cache or to some database entry or wherever we would want to keep this. And then display this on a dashboard saying we have these many failed messages for this account, for this section, for this integration. That could be fine. That would give you a good overview for you to take action on the day after."

**Advantages:**
- Real-time visibility into failed messages by account/section/integration
- Enables proactive decision-making
- Can be extended to gather actionable items (permission errors, API key rotations, 401s)

**Implementation note:** [Erik Andersson] suggested filtering errors into actionable categories that could be escalated to SOC immediately (permission/API key issues) versus requiring CRM developer investigation.

### Option 2: Existing Logging and Query-Based Approach

[Michal Rosikiewicz] described their current process:

> "What we do is we be weekly after the on call week we verify logs with additional queries in logging sites and then based on the results we either take some action that we should plan fix for us ourselves or just inform customer that something is failing and they have to fix the process."

This relies on **log insights queries** (execution time: ~12 hours) to identify issues post-facto. However, this requires discipline and doesn't provide real-time visibility.

### Required Redesign Trade-offs

[Erik Andersson]: "Either you need to be on top of checking these errors and taking whatever action you can and still rely on getting the notification from the alarms and convert some errors to warnings in some cases when you see it's warranted, or you shift this over to some other like DevOps service that you might already be utilizing."

The team agreed that **changing the retry delay directly relates to monitoring decisions**: shorter retries require more aggressive monitoring to catch and remediate issues before queue clogging occurs.

---

## Retry Logic: Current State and Problems

### Current Exponential Backoff Implementation

The current retry strategy uses **8 maximum retries** with exponential backoff:

```
15 seconds → 15 seconds → 1 minute → 5 minutes → 15 minutes → 1 hour → 4 hours → 6 hours
```

**Total retry time: approximately 1.5 days**

### The Implementation Problem

[Erik Andersson] identified a critical code smell in the existing backoff function:

> "This function should be obliterated from orbit because it is so ugly. In SQS you have metadata showing how many times you've retried a message, but instead what this function does is it calculates what is the back off time and then we compare if the back off time is longer than something. Then we set the retry multiplier based on how long it has been in retry instead of utilizing the already existing metadata on how many times has this message been retried."

The function **calculates elapsed time** to determine backoff instead of using SQS's native **`ApproximateReceiveCount`** metadata field.

### Why This Matters for CRM Downtime Scenarios

[Michal Rosikiewicz] raised a critical concern:

> "If CRM is not available for 20 minutes, they have some downtime. For example, right now we would try for like 4 or 5 hours. So finally when they get up, they will receive those messages. If we reduce it to 15 minutes, they won't receive this message, yeah."

**The trade-off:** Shorter retry windows may cause message loss during legitimate CRM maintenance windows, but longer windows contribute to queue clogging and alert desensitization.

---

## Redesigned Retry Strategy

### Decision: Reduce Total Retry Time to ~5 Hours

The team agreed that **if an issue isn't fixable within 4-5 hours, it shouldn't continue retrying for another 1+ day**.

[Erik Andersson]: "If someone is not able to fix this in like 4 hours total then we shouldn't care about it for next one day."

### New Backoff Schedule

**Proposed interval:** 15 minutes between retries (configurable as a starting point)
**Maximum retries:** ~20 (to achieve approximately 5 hours total: 5 hours = 300 minutes ÷ 15 minutes = 20 retry attempts)

[Erik Andersson]: "Let's say that we have 60 retries, that seems like an absurd amount. Let's say 75 then, so then it would continue for... [calculating] 75 × 15 minutes = 1125 minutes, which is about 18+ hours. Let's target 20 retries to stay within 5 hours."

The default case after exhausting retry attempts would return **15 minutes** continuously until the maximum is reached.

### Refactored Function Design

**Current ugly approach:**
```
Calculate elapsed time → Compare elapsed time → Set multiplier based on time
```

**Proposed clean approach:**
```go
Switch on ApproximateReceiveCount (retry count)
  Case 0: return 5 seconds
  Case 1: return 15 seconds
  Case 2: return 1 minute
  Case 3: return 5 minutes
  Case 4-19: return 15 minutes
  Default: return 15 minutes
```

[Erik Andersson]: "When we are calling it, we have the SQS message, so we have access to the message metadata. We base it on that. Because where we call this function we have access to the message. We are iterating over each message, so we have access to the message metadata."

**Function signature change required:**
- Add the SQS message as input parameter (currently missing)
- Extract `ApproximateReceiveCount` from message metadata
- Use switch/case on the count value instead of elapsed time calculation

### Where Changes Apply

The backoff logic is used in two critical places:
1. **Sluice Worker** — when full syncs are running and messages need backoff
2. **Outbound Worker** — when CRM returns errors

Both will benefit from this cleaner implementation.

### Why This Is a Good Change

[Erik Andersson]: "That is the easy part that I think we three can just agree on and create the story from the get go. This change is a very easy one."

- **Simplification:** Removes time-based calculation logic
- **Uses native metadata:** Leverages SQS's built-in `ApproximateReceiveCount`
- **Configurability:** Easy to adjust retry counts and intervals without rebuilding time calculations
- **Visibility:** Makes retry attempt count explicit in code
- **Maintainability:** Switch/case is cleaner than conditional time comparisons

---

## Testing Strategy: Mock CRM Service

### Why Manual Testing Is Necessary

[Michal Rosikiewicz] asked the critical question: "So you would like to broke something in the CRM site, remove the key or because I think you want to verify if after four hours it's gone, yeah."

To safely test failure scenarios without breaking actual customer CRM systems, the team has access to a **mock service** that simulates CRM behavior.

### Mock Service Architecture

**Location:** Customizable mock CRM service available in development/staging environments

**How it works:**
- Service implements the **generic connector contract**
- Can be configured to simulate any CRM system (E-deal, Efficy Enterprise, Maxo, etc.)
- Returns predictable test data on successful requests
- Can return controlled errors on demand

**Installation method:**
```
POST to mock service URL with:
  - Environment: staging (or development)
  - CRM Type identifier in API key (FSC_CORPORATE, E_DEAL, MAXO, etc.)
  - Returns fake but functional CRM installation
```

### Identifying Mock Installations

[Erik Andersson] explained how to spot a mock CRM:

> "The mock service is configured to say it appends the entity name before the attribute. So here you see all of the attributes available are like person, person, person, person, person, person, person. That's how I know that this is a fake CRM."

**Example:**
- **Mock CRM (E-deal):** All attributes prefixed with "person_" (person_email, person_firstname, etc.)
- **Real CRM:** Actual field names (email, firstname, etc.)

### Mock Service Capabilities and Limitations

**Supports:**
- Field mapping configuration
- Sync conditions
- Consent messages
- Sending email/SMS/form/tool events
- Full sync operations
- Returns predictable data: "firstname_1", "firstname_2", etc. (incrementing values)
- Event posting with mock responses and random UIDs

**Does not support (yet):**
- Real-time messages from mock CRM back to Apsis (possible to implement but not yet done)

### Testing the Backoff Change

**Approach:**
1. Deploy a change to the mock service to return errors for specific account/section combinations
2. **Don't include the mock change in the main pull request** — deploy it separately
3. Manually configure the mock to return HTTP 500 errors for test integration
4. Trigger messages to the outbound worker and verify:
   - Messages retry with new backoff schedule
   - Messages are discarded after ~5 hours (20 retries at 15-minute intervals)
   - No messages remain stuck in the queue

**Mock Service Code Location:**
- File: `echo_server.go` (or similar in mock service repo)
- Function: `PostCampaignEvents` or equivalent

**Code pattern to add:**
```go
if body.Account == "test_account" && body.Section == "test_section" {
  return errors.New(500, "Internal Server Error")
}
```

**Why use conditionals:** "This prevents us from breaking every other mock integration. You can filter it on the account and section so you eliminate that risk."

### Deployment Process

[Erik Andersson]: "You can push this manually by going to mock service, doing your change and then deploying this manually outside of your branch, so you don't incorporate it in the pull request."

This allows testing the backoff logic without committing test error code to the main codebase.

---

## Database and Error Record Keeping

### Historical Error Logging

When messages are discarded or heavily retried, the team logs all activity for audit purposes:

[Erik Andersson]: "We typically log everything. So we like we don't completely lose the messages. You could in theory send them like empty the dead letter queue and then send every message to the dead letter queue and redrive them later on, depending on how you want to handle it."

This ensures that even if messages are dropped from the active queue to prevent system-wide clogging, they can be recovered and redriven if needed.

---

## Queue Emergency Procedures: Message Dropping

### When Message Dropping Is Necessary

In rare cases where a single customer's misconfiguration causes massive queue clogging, the team has an emergency procedure to drop messages for that customer only:

[Erik Andersson]: "Let's say now that someone did a big email sending event, something like that one time when they sent like 500,000 events or something in one go and for whatever reason the outbound worker gets stuck there, so like their 500,000 messages block everything else. In that case, what we have had to do historically is to deploy a fix to production where we essentially discard that whole sending."

**Rationale:** "It's better that like those events are not sent rather than us not sending anything for any customer for like potentially days because that's not going to be a manageable situation."

### Implementation: Message Processor Filtering

**Location:** `message_processor.go` in the outbound worker

**Mechanism:** In the `ProcessMessage` function, add conditional logic:

```go
if ik.Account == "problematic_account" && ik.Section == "problematic_section" {
  logger.Warn("Skipping message for account/section due to clogged queue", fields)
  return nil  // Return success without processing
}
```

**Key point:** Returning `nil` (without error) is interpreted as "message successfully processed," so the message is removed from the queue immediately.

**Speed:** This allows clearing tens of thousands of messages from the queue **within a minute** instead of waiting for retries to complete.

### Prerequisites and Cautions

- Messages must be readable from the queue (not in backoff timeout phase)
- This procedure should **never be done without explicit approval**
- All dropped messages are logged and can be retrieved from dead letter queue for later replay
- Requires coordination with team before deployment to production

[Michal Rosikiewicz]: "Will you give me approval on this?"
[Erik Andersson]: "There are too many alarms to just let us discard all messages when they come instead."

The emergency approval should come from the team lead or on-call manager.

---

## High CPU Alarm Issues in Delta Sync Worker and Managers

### Recurring Alerts

[Michal Rosikiewicz] reported: "I'm bothered during this meeting the third time with high CPU in the Delta Sync worker."

This is **not coincidental** — it's related to message volume cascading from other services.

### Cascade Pattern: Delta Sync → Sluice Worker

[Erik Andersson]: "Those two services kind of go hand in hand. First the Delta Sync worker is bombarded. If someone does a massive sync, even though we said you shouldn't do it because they changed their metadata structure or something and then every contact they have in the whole CRM system is updated, from the Delta Sync worker or this Delta Sync manager they are moved to the sluice worker. So if you get it in the Delta Sync worker you're very high risk to get it in the sluice worker as well."

### Current Alarm Configuration

The CPU alarms are overly sensitive. Example:
- **Threshold:** CPU > 94% for only **1 evaluation period (300 seconds)**
- **Problem:** Legitimate bursts trigger alerts; actual sustained overload is not distinguished

### Proposed Adjustment

[Michal Rosikiewicz]: "Usually do like 5 consecutive periods of like 60 seconds each, so it means that 5 minutes. We found it works best at about 55 minutes for us."

**Recommended:** Change alarm to require **5 consecutive evaluation periods** (each 5 minutes = 25 minutes total) before triggering, or align with historical data showing when false positives occur.

[Tomasz Kowalski]: "We need to look how the alarm is talking and adjust accordingly."

### Cross-Service Applicability

The Delta Sync manager is centrally configured in the **shared manager CloudFormation template**, so this change would affect **all managers**, not just Delta Sync.

[Erik Andersson]: "There would not be an issue to change this for all managers, I would say. There is no issue if you get a sudden burst. It is an issue if that persists."

---

## Summary of Design Decisions and Action Items

### Agreed Changes

1. **Refactor exponential backoff function** (2-day task):
   - Use `ApproximateReceiveCount` metadata instead of elapsed time calculation
   - Implement clean switch/case based on retry count
   - New schedule: 15-minute intervals, ~20 retries, ~5-hour maximum

2. **Testing with mock service** (part of above task):
   - Deploy error-returning mock to staging
   - Verify messages retry correctly and are discarded after 5 hours
   - Verify no messages get stuck in the queue

3. **Monitoring and dashboard creation** (separate, larger task):
   - Build metrics from failed message events
   - Create AWS CloudWatch dashboard showing failed messages per account/section/integration
   - Scope: Larger discussion, may be done in follow-up session with Lukasz

4. **High CPU alarm tuning**:
   - Change Delta Sync manager CPU alarm to require 5 consecutive evaluation periods
   - Use 5-minute periods (25 minutes total before triggering)
   - Adjust based on historical data patterns

### Unresolved Questions / Future Work

1. **Monitoring strategy finalization:** Should issues be handled via:
   - Daily dashboard review by on-call team?
   - Real-time SOC escalation for certain error types?
   - Continued weekly log analysis approach?
   - Hybrid approach combining approaches?

2. **Dead letter queue strategy:** How should the team handle messages that exhaust retries?
   - Should they be automatically sent to DLQ?
   - How often should DLQ be reviewed and redriven?

3. **CRM downtime recovery:** Are 5 hours sufficient for typical CRM maintenance windows?
   - May need to increase retry period based on SLA agreements with specific CRM customers

4. **Mock service enhancement:** Should real-time message support be added to mock service for more complete testing scenarios?

---

## Key Takeaways

1. **Current retry strategy is inefficient:** 1.5-day retry window with time-based backoff calculation is overly complex and contributes to alert desensitization.

2. **Visibility is essential before shortening retries:** Reducing retry time to 5 hours requires corresponding investment in monitoring (dashboards, metrics, escalation procedures) to catch issues that can't self-heal.

3. **Queue clogging is a real risk:** Single-customer misconfiguration can block the entire queue for all customers. Emergency message-dropping procedures are necessary but must be used carefully.

4. **Clean code enables quick pivots:** Refactoring the backoff function to use native SQS metadata makes it trivial to adjust retry schedules without complex time calculations.

5. **Mock service is underutilized:** A functional mock CRM exists for testing error scenarios safely without affecting customers.

6. **Alarm tuning needs historical validation:** High CPU alarms are generating false positives. Adjust thresholds based on actual burst patterns from log data.

7. **Monitoring should be proactive, not reactive:** Moving from weekly log reviews to real-time dashboards enables faster issue detection and prevents queue clogging scenarios.