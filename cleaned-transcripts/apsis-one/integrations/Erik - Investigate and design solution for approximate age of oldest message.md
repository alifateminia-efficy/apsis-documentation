---
source_file: Erik - Investigate and design solution for approximate age of oldest message.txt
domain: Apsis One Integrations
topics: 
  - Page duty alarm philosophy and auto-resolution
  - Outbound worker queue management and CRM error handling
  - Message retry logic and exponential backoff optimization
  - Queue clogging and message prioritization in FIFO queues
  - Monitoring dashboard design for actionable errors
  - Mock CRM service for testing
  - Message discarding strategies for queue recovery
  - CPU alarm configuration for manager services
speakers: 
  - Erik Andersson
  - Michal Rosikiewicz
  - Tomasz Kowalski
key_components:
  - Outbound worker
  - Sluice worker
  - Delta Sync manager
  - SQS message queue
  - Mock CRM service
  - Message processor
  - Exponential backoff function
  - AWS CloudFormation (SAM file)
session_type: architecture-review
subdomains:
  - Architecture
  - Microsoft Dynamics Integration
---

## Session Overview

This session focused on designing solutions for managing message age and queue health in the Apsis One Integrations platform, specifically addressing how to handle continuous errors in the outbound worker that prevent alarm auto-resolution. The team discussed the fundamental issue that the current 4-hour retry window with exponential backoff is both too aggressive (causing queue clogging) and too passive (not surfacing failure volume). The discussion covered retry logic refactoring, monitoring strategy design, leveraging mock CRM environments for testing, and emergency queue recovery procedures.

---

## Page Duty Alarm Philosophy and Design

### Current Alarm Behavior

[Erik Andersson]: The fundamental design principle is that alarms should auto-resolve after a configured time period without new errors. This works well for one-off failures, but the problem arises with **continuous errors from specific customers**.

> Whenever there is an actual error in the systems, the alarm will trigger and if there has not been any more errors for a given time then it will auto-resolve. If the same issue occurs again, hopefully you would have actually done something about it.

The auto-resolution mechanism prevents alert fatigue but creates a blind spot: when errors persist continuously, the alarm stays in alert state and we lose visibility into **error volume and frequency**.

### Error Categories and Ownership

**Manager services** (customer-facing integrations) errors typically fall into two buckets:
1. **Internal errors**: Bugs in Apsis code or malformed customer data — Apsis team owns the fix
2. **External errors**: CRM system failures, permission issues, API key rotations — Customer/CRM team owns the fix

For the **outbound worker**, nearly all errors are CRM-related:
- CRM returns HTTP errors (5xx, permission denials, validation failures)
- Apsis cannot directly fix CRM system issues
- The team can only notify relevant parties (SOC for permission issues, CRM developers for bugs)

### The Queue Clogging Problem

[Erik Andersson]: The outbound worker operates as a **FIFO queue** with a critical constraint: ABS (the processing engine) only scans the first 20,000 messages in the queue for message groups.

> If you have 20,000 messages in the queue for the same customer for the same type — for example, they are blocking an e-mail sending events from us and we have 20,000 events from the same customer — then any message that is behind those 20,000 messages will not be processed because ABS only looks for message group in these first 20,000 messages and they are laying there being timed out.

This creates a cascading failure: a single misconfigured customer can block all other customers' messages from being processed until those 20,000 messages are resolved or discarded.

**Cannot ignore this:** The team cannot simply convert errors to warnings and ignore them, as persistent queue buildup will eventually block all integrations.

---

## Current Retry Logic and Its Problems

### Exponential Backoff Implementation

The current retry strategy uses exponential backoff with this sequence:
```
Attempt 1: 15 seconds
Attempt 2: 15 seconds
Attempt 3: 1 minute
Attempt 4: 5 minutes
Attempt 5: 15 minutes
Attempt 6: 1 hour
Attempt 7: 4 hours
Attempt 8: 6 hours (then message is discarded)
```

**Total retry duration: approximately 1.5 days**

### Design Flaw in Current Implementation

[Erik Andersson]: The current implementation calculates backoff delay by **computing elapsed time on the message**, rather than using SQS metadata.

> This function should be obliterated from orbit because it is so ugly. In SQS you have metadata on how many times you've retried a message. Instead, this function calculates what the backoff time is and then compares if the backoff time is longer than something, then we set the retry multiplier based on how long it has been in retry instead of utilizing the already existing metadata.

The function checks timestamps to determine backoff instead of using `approximate_number_of_receives` from the SQS message metadata—this is unnecessarily complex and brittle.

### The Problem with 4-Hour Retry Windows

[Michal Rosikiewicz]: During customer CRM downtime, the long retry window has a paradoxical effect:

> If CRM is not available for 20 minutes and we try for 4 or 5 hours, when they get up they will receive those messages. But if we reduce it to 15 minutes, they won't receive the message.

[Erik Andersson] counters: The solution is to keep total retry time reasonable (4-5 hours) but **decrease the interval between retries**:

> We need to compensate by increasing the retry times. The retrying is going on for like 1.5 days essentially. If someone is not able to fix this in like 4 hours total then we shouldn't care about it for the next one day.

---

## Proposed Retry Logic Redesign

### New Specification

**Total retry duration:** 5 hours maximum (configurable)
**Retry interval:** 15 minutes (configurable starting point)
**Number of attempts:** 75 retries

Using 5 hours ÷ 15 minutes = 20 discrete retry intervals. At 75 retries with final intervals at 15 minutes, this covers approximately 5+ hours before message discard.

### Implementation Strategy

Replace the ugly elapsed-time calculation with a clean switch/case based on retry count:

```go
func selectTimeout(message *sqs.Message) time.Duration {
  switch message.Attributes["ApproximateReceiveCount"] {
    case 0:
      return 5 * time.Second
    case 1:
      return 15 * time.Second
    case 2:
      return 1 * time.Minute
    case 3:
      return 5 * time.Minute
    case 4:
      return 15 * time.Minute
    default:
      return 15 * time.Minute  // Cap at 15 minutes for all retries beyond attempt 4
  }
}
```

**Key improvements:**
1. Uses native SQS metadata (`ApproximateReceiveCount`) instead of timestamp calculations
2. Much simpler, more maintainable code
3. Easy to adjust intervals in one place
4. No magic number calculations

### Where to Apply This

Changes needed in two worker processes:
1. **Outbound worker**: Primary use case — retrying failed CRM posts
2. **Sluice worker**: Retrying during full sync backoffs

Both workers call similar backoff logic through the message processor.

---

## Error Monitoring and Actionability

### Current Problem

Alarms trigger but don't show **what** is failing or **in what volume**. The team discovers actual issues through:
- Weekly log analysis after on-call weeks
- Manual log queries in log analysis tools
- Customer complaints

### Proposed Solution: Monitoring Dashboard

[Erik Andersson]: Create an AWS dashboard that aggregates actionable errors per customer/integration.

> You could create a metric from this and put it on a dashboard in AWS and it would be a first step. If you have filters like this already in your services, we can incorporate that into integration and you can do an export of this whenever you want to because this means you then know you need to notify the developers and they can hopefully take some kind of action.

**Dashboard content:**
- Failed message count per customer per integration type
- Error categorization:
  - Permission errors (401/403) — requires SOC/customer action
  - API key errors — requires customer to rotate/update credentials
  - SQL/database errors — requires CRM developer investigation
  - Transient 5xx errors — may auto-resolve
- Filtered view: Personal accounts (starting with "PA") typically ignored unless causing queue clogging

**Benefits:**
1. On-call engineer can check dashboard day-after to see what occurred overnight
2. Enables quick triage: categorize error type → route to correct team
3. Separates **critical issues** (queue clogging) from **notice items** (isolated errors)

### Log Analysis and Error Filtering

Example error encountered during testing:

```
Bad request - SQL error: Missing required fields in test environment
```

[Erik Andersson]: This was a failed database migration. Error was visible in logs, but the team couldn't clear it from the queue due to 4-hour retry delays.

> We saw this error in the logs, so we knew exactly what the problem was. He fixed this in real time, but because we were putting a 4-hour retry on this, we couldn't continue with the testing because I couldn't clear this message from the queue. I can't purge the queue in production for obvious reasons.

By reducing retry intervals to 15 minutes, errors can be tested and verified cleared much faster.

---

## Mock CRM Service for Testing

### What is the Mock Service?

The mock CRM service is a **customizable, controllable CRM environment** that implements the generic connector contract. It allows testing against a "fake CRM" without affecting real customer instances.

[Erik Andersson]: Key location and usage:

> You have this service here which is essentially fulfilling the generic connector contract. So this is a CRM we control, like a customizable CRM system. It can return whatever we want.

### How to Identify a Mock Instance

[Erik Andersson]: Mock instances append the entity name to all attributes:

> The mock service is configured to say it appends the entity name before the attribute. So here you see... all of the attributes available are like person, person, person, person... That's how I know that this is a fake CRM in comparison to real instances.

In real CRM systems (E-deal, Efficy, etc.), attribute names vary by entity type and are more diverse.

### Setting Up a Mock Installation

To install a mock CRM pretending to be a specific system:

1. **URL**: Point to the mock service endpoint
2. **API key**: Specify which CRM system to simulate
   - `fscu` → Maxo (uses `IMartic profile` entity)
   - `fsccorporate` → FSC Corporate (uses `person` entity)
   - `edeal` → E-deal (uses `person` entity)

Example:
```
POST /install
URL: <mock-service-url>
API_KEY: fscu
```

This creates a staging instance that pretends to be Maxo, complete with predictable data generation.

### Mock Service Capabilities

**Supported:**
- Field mapping configuration
- Sync condition setup
- Consent message handling
- All outbound event types (email, SMS, form, tool events)
- Full sync operations (returns predictable, repeatable data)
- Random entity ID generation ("Trust me bro")

**Not supported (yet):**
- Real-time messages from mock service back to Apsis
- (Could be implemented but hasn't been)

### Using Mock for Testing Retry Logic

To test the new backoff logic:

1. Configure mock to return errors for specific account/section combinations
2. Deploy mock changes manually (outside PR/branch)
3. Send messages to the mock instance
4. Verify backoff timers work correctly
5. Confirm messages are retried at expected intervals and discarded after 5 hours

**Example modification** (in `server.go` or equivalent mock handler):

```go
if request.Account == "test_account" && request.Section == "test_section" {
  return HTTP 500 "Internal Server Error"
}
```

This approach:
- Doesn't break real CRM integrations
- Allows repeatable, controlled testing
- Doesn't require breaking a customer's actual CRM system

---

## Emergency Queue Recovery: Message Discarding

### When Queue Recovery is Necessary

[Erik Andersson]: Severe scenarios where a single customer's misconfiguration blocks all other customers:

> Let's say someone did a big e-mail sending. They sent like 500,000 events or something in one go and for whatever reason the outbound worker gets stuck there, so their 500,000 messages block everything else. In that case, what we have had to do historically is deploy a fix to production where we essentially discard that whole sending.

> It's better that those events are not sent rather than us not sending anything for any customer for like potentially days because that's not going to be a manageable situation.

This is a **last resort** but preferable to cascading failure across all customers.

### How to Implement Message Discarding

In the **message processor** (core processing engine), check integration/account/section and return early without processing:

```go
if processMessage(integrationKey, message) {
  // In message processor function:
  if integrationKey.Account == "problematic_account" && 
     integrationKey.Section == problematic_section {
    logger.Warn("Skipping message due to clogged queue", "account", integrationKey.Account)
    return nil  // Success return = message removed from queue
  }
  
  // Normal processing continues below
  sendToCRM(...)
}
```

**Critical detail**: Returning `nil` (no error) is interpreted as **successful processing**, so the message is removed from the queue without being sent.

### Speed of Queue Recovery

[Erik Andersson]: Discarding thousands of messages is extremely fast:

> If you do it like this, you will empty like 10s of thousands of messages in the queue within like a minute. It is ridiculously fast.

### Data Loss Mitigation

While discarding is destructive, the team can:
1. **Log every discarded message** — no data is truly lost, can be audited
2. **Send to dead letter queue** — messages can be driven back later after issue is resolved
3. **Get approval first** — this is a human decision, not automatic

> This should not be taken lightly, but it's still preferable to do if some customer has done something bad and now nothing is working for any customer.

---

## Queue Visibility and Message Age Detection

### Why Message Age Matters

The session title references "approximate age of oldest message" — this is a proxy for queue health.

**Implication**: If the oldest message in the outbound queue is 6+ hours old, it's likely stuck in retry and either:
1. Will be discarded soon anyway
2. Indicates a customer blocking all downstream traffic
3. Should trigger investigation

By shortening retry windows from 1.5 days to 5 hours, **stale messages are cleared faster**, improving queue throughput and reducing customer impact.

---

## CPU Alarms for Manager Services

### Recurring False Alarms

[Michal Rosikiewicz]: The Delta Sync manager has triggered high-CPU alarms three times during this session, suggesting misconfigured thresholds.

> I'm bothered during this meeting the third time with high CPU in the Delta Sync worker.

### Root Cause

These services experience **legitimate spikes** when customers perform massive syncs (e.g., updating all contact metadata), but the current alarm is configured too sensitively to distinguish between:
- Brief spikes (normal, expected)
- Sustained high CPU (actual problem)

### Proposed Fix

[Erik Andersson]: Adjust alarm evaluation periods to require sustained CPU usage:

> Something weird is going on... if it is over an hour or something like that. It should not be one minute, it should be like if it is over 94, like over an hour or something like that.

**Better configuration** (suggested by Michal):
- **5 consecutive evaluation periods of 60 seconds** = alarm only fires if high CPU persists for 5+ minutes
- Or: **Longer evaluation window** (e.g., 25 minutes) before triggering alert

This prevents one-off spikes from waking on-call engineers while still catching genuine sustained overload.

### Configuration Locations

- **Sluice worker CPU alarm**: Configured in the sluice worker's SAM file (isolated)
- **Delta Sync manager CPU alarm**: Configured in the **centralized manager CloudFormation template** (affects all managers)

Changing the manager template impacts all manager services, so should be validated against historical data first.

---

## Summary of Action Items and Decisions

### Retry Logic Refactoring (2-point story)

**Title**: Refactor exponential backoff to use SQS metadata and reduce retry window

**Details**:
- Replace elapsed-time calculation with switch/case on `ApproximateReceiveCount`
- Reduce total retry duration from 1.5 days to 5 hours
- Keep initial interval at 15 minutes (configurable)
- Increase max retries to ~75 attempts
- Apply to both outbound worker and sluice worker

**Testing approach**:
- Use mock CRM service configured to return HTTP 500 for specific test account/section
- Deploy mock changes manually (outside PR)
- Verify backoff timers and message discard after 5 hours
- Merge changes with confidence

**Effort**: 2 days (implementation + testing in both workers)

### Mock Service Enhancement (part of testing strategy)

**Details**:
- Modify mock to return configurable errors for specific account/section combinations
- Example: `POST_CAMPAIGN_EVENTS` handler returns 500 for test_account/test_section
- Use `if` conditions to avoid breaking other mock integrations
- Deploy manually to staging

### Monitoring Dashboard Design (larger initiative, defer detail discussion)

**Approach**: Defer detailed design until Lukasz (presumably product/monitoring lead) is available

**Preliminary scope**:
- Aggregate failed messages per customer per integration
- Categorize errors by type (permission, API key, database, transient)
- Filter personal accounts (PA prefix) by default
- Surface actionable metrics for on-call triage

### CPU Alarm Tuning

**Action**: Review Delta Sync manager and sluice worker CPU alarm configurations

**Change**: Extend evaluation period to 5+ consecutive periods of 60 seconds (or 25+ minute sustained) before firing alert

**Validation**: Compare against historical data to ensure this doesn't mask real outages

---

## Key Takeaways

1. **Queue clogging is a multi-customer problem**: One misconfigured customer can block all others if 20,000 messages accumulate. This isn't ignorable as a warning-level issue.

2. **Current retry logic is both wrong and ugly**: The 1.5-day total retry duration is excessive, and calculating backoff from elapsed time is unmaintainable. Use SQS metadata instead.

3. **5-hour retry window + 15-minute intervals balances resilience and responsiveness**: CRM downtime is usually resolved within 4 hours. After 5 hours, if still failing, the issue requires human intervention anyway.

4. **Mock CRM service is underutilized**: The team has a powerful testing tool but hasn't fully leveraged it for regression testing. Use it for backoff testing and other edge cases.

5. **Monitoring gaps hide failures**: Weekly log analysis after on-call weeks is reactive. A dashboard showing failed messages per customer would enable proactive triage and faster incident response.

6. **Message discarding is a valid last resort**: When queue clogging threatens all customers, discarding messages is preferable to cascading failure—as long as they're logged and the action is approved.

7. **Alarm sensitivity needs tuning**: False alarms on CPU (from normal spikes) reduce signal. Extend evaluation windows to detect sustained problems, not brief bursts.

---

## Unresolved Questions and Deferred Discussions

1. **Monitoring dashboard detailed design** — Deferred until Lukasz is available. Covers: metrics definition, alert routing logic, SLA integration, customer-facing vs. internal views.

2. **Error categorization and routing** — How should errors be automatically routed to SOC vs. CRM development vs. customer? Requires process/organizational decisions, not just technical implementation.

3. **Personal account handling** — Should personal test accounts (PA prefix) be completely ignored, or are there scenarios where they should trigger alerts? Currently ignored unless causing queue clogging.

4. **Dead letter queue strategy** — If messages are discarded, should they go to a dead letter queue for later investigation/retry? How long to retain? Who reviews?

5. **CPU alarm thresholds for all managers** — Changing the centralized CloudFormation template affects all managers. Need to validate safe thresholds across all manager types and workloads.