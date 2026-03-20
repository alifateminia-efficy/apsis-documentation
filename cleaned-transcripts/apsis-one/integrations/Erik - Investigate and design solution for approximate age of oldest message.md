---
source_file: Erik - Investigate and design solution for approximate age of oldest message.txt
domain: Apsis One - Integrations
topics: [Alarm Design Philosophy, Message Queue Management, Retry Logic and Backoff Strategy, Queue Monitoring and Dashboards, Mock CRM Testing Environment, Message Processing and Queue Clearing, CPU Alarms and Alert Tuning]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Outbound Worker, SQS Queue, Delta Sync Worker, Sluice Worker, Mock Service, Message Processor, CRM Integration, Exponential Backoff Function]
session_type: architecture-review
---

## Session Overview

This knowledge transfer session covers the design philosophy and implementation strategy for handling errors and message queue management in the Apsis One Integrations platform. The primary focus is on redesigning the retry/backoff mechanism for the outbound worker to reduce total retry time from approximately 1.5 days to 5 hours, implementing queue monitoring dashboards, and establishing a testing approach using a mock CRM service. The team also discusses alarm tuning strategies to reduce false positives in CPU monitoring.

---

## Alarm Design Philosophy and Auto-Resolution Strategy

### Current Alert Handling Model

[Erik Andersson]: The design philosophy for page duty alarms operates on a principle of auto-resolution. When an actual error occurs in the system, the alarm triggers. If no additional errors occur within a given timeframe, the alarm automatically resolves. If the same issue recurs, it triggers again. This ensures that operators only remain paged until either:
1. Action is taken to fix the underlying issue, or
2. The error stops recurring naturally

### The Problem with Continuous Errors

[Erik Andersson]: The system works well for intermittent errors, but breaks down when customers experience continuous, uninterrupted failures. In this scenario, the alarm never auto-resolves because errors keep occurring. The alert remains in triggered state indefinitely, masking the true volume of failures occurring in the queue. An operator cannot see the actual number of failed messages—they only know the alarm is active.

> "The problem here now is that there are continuously errors for some customers. So this like never auto resolves itself mean like it just trigger and then there can be as many errors as humanly possible, but we don't know that because it is still in the alert stage because it has not yet auto resolved, which might not be the most optimal of the situations."

### Error Classification by Source

**Manager Services Errors** (customer-facing integrations):
- Triggered by customer actions: installation, setup, full sync operations
- Often caused by customer error: malformed data, unexpected configurations
- Actionable by the Apsis One team

**Outbound Worker Errors** (CRM integration):
- Almost always connected to the CRM system itself
- Not directly actionable by Apsis One team—require CRM vendor action
- Examples: permission issues, API key rotation, unexpected CRM behavior
- Critical constraint: Cannot manually fix issues on the customer's CRM system

### Queue Depth Risk Management

[Erik Andersson]: The outbound worker queue is a FIFO (First In, First Out) queue with a hard limit of 20,000 messages. This limit creates a cascading failure risk:

> "If we cross this 20,000 messages in the queue limit for too long of a time. We might risk ending up in some nasty situation because of how AVS looks for the message groups to process."

**The AVS Limitation**: AVS (the queue processing system) only searches for message groups within the first 20,000 messages in the queue. If the first 20,000 messages are all from one customer who is experiencing failures (e.g., 20,000 blocking email sending events), any message from other customers positioned after those 20,000 will not be processed because AVS will never reach them.

**Impact**: This can clog the entire outbound worker queue, preventing message delivery for all customers, not just the problematic one.

---

## Monitoring and Dashboard Strategy

### Current Post-Incident Review Process

[Michal Rosikiewicz]: The current approach relies on manual review after the on-call week ends. The team runs additional log queries and investigates which errors require:
1. Internal fixes that the development team should implement, or
2. Customer notifications about their own process failures

### Proposed Monitoring Enhancement

[Erik Andersson]: Instead of relying solely on alarms that may never resolve, the solution is to build a **failure tracking system**:

1. **Statistics Cache or Database Entry**: When a message fails to process in the outbound worker, record it in a persistent data structure
2. **Aggregation by Dimension**: Group failures by:
   - Account
   - Section
   - Integration type
3. **Dashboard Visualization**: Display actionable items on a centralized dashboard
4. **On-Demand Review**: Allow the on-call team to check the dashboard daily and determine what action is needed

### Expanded Dashboard Scope

Beyond outbound worker failures, the dashboard could surface other actionable errors across the system:
- **401 Unauthorized errors**: Indicate API key rotation; notify SOC
- **Permission errors**: When retrieving data; indicate customer-side permission issues
- **CRM connectivity failures**: Different handling based on whether it's a permission vs. availability issue

> "If you have such a dashboard, you could also add errors like if you're getting permission problems when you're trying to retrieve data or something in the anywhere in the system, or if you're getting like a 401 that's some API key has been rotated and it you can no longer reach the you can no longer reach the in stuff like you could in theory gather all such actionable items on that dashboard per customer"

### Implementation Option

[Michal Rosikiewicz]: The team can leverage AWS CloudWatch to create metrics and display them on a dashboard. This is identified as the first step in implementing enhanced monitoring.

---

## Retry Logic and Backoff Strategy Redesign

### Current Implementation Problem

The existing retry mechanism uses an exponential backoff function that calculates delays based on **how long the message has been in the retry queue**, rather than **how many times it has been retried**.

**Current Exponential Backoff Sequence** (approximately):
- Attempt 1: 15 seconds
- Attempt 2: 15 seconds
- Attempt 3: 1 minute
- Attempt 4: 5 minutes
- Attempt 5: 15 minutes
- Attempt 6: 1 hour
- Attempt 7: 4 hours
- Attempt 8: 6 hours
- Total: ~18 hours (approximately 1.5 days)

[Erik Andersson]: The implementation is problematic:

> "This function should be obliterated from orbit because it is so ugly."

The function calculates backoff time by comparing elapsed time against hardcoded thresholds, rather than using SQS's built-in metadata that tracks the number of retries for each message.

```
// Current approach (BAD):
Compare elapsed_time > some_threshold
  If true: set retry_multiplier based on elapsed time
  
// Should be (GOOD):
Use SQS's ApproximateReceiveCount metadata
  Switch on retry count and return appropriate delay
```

### Redesigned Retry Strategy

**Goal**: Reduce total retry window from 18 hours to approximately 5 hours, while allowing sufficient time for CRM systems to recover from temporary outages.

**New Backoff Sequence** (proposed):
```
Retry Attempt 0: 5 seconds
Retry Attempt 1: 30 seconds
Retry Attempt 2: 1 minute
Retry Attempt 3: 5 minutes
Retry Attempt 4: 15 minutes
Retry Attempt 5+: 15 minutes (plateau)
```

**Rationale for 5-Hour Window**:
[Erik Andersson]: If a CRM system experiences an outage or issue, the typical on-call window is approximately 4-5 hours. If an issue is not resolved within that window, continuing to retry for another 18 hours is not productive. The on-call team should have taken action by then.

> "If someone is not able to fix this in like 4 hours total then we shouldn't care about it for next one day."

**Calculation of Retry Attempts**:
- 5 hours = 300 minutes
- Base interval: 15 minutes
- Retry attempts needed: 300 / 15 = 20 retries minimum
- **Proposed: 75 retries maximum** (provides sufficient headroom while capping at 5 hours)

[Michal Rosikiewicz]: This change also solves the queue clearing problem:

> "That's that's the thing that I think we should definitely fix because. If if someone is not able to to fix this in like 4 hours total then we shouldn't care about it for next one day."

### Implementation: Refactored Backoff Function

**File to modify**:
```
lib/sqs.go
Function: select_timeout (or similar)
```

**New Implementation Approach**:

Instead of calculating based on elapsed time, use SQS message metadata:

```go
// Function signature change:
// FROM: selectTimeout(currentTime time.Time) time.Duration
// TO: selectTimeout(message *SQSMessage) time.Duration

// Within function:
switch message.ApproximateReceiveCount {
  case 0:
    return 5 * time.Second
  case 1:
    return 30 * time.Second
  case 2:
    return 1 * time.Minute
  case 3:
    return 5 * time.Minute
  case 4:
    return 15 * time.Minute
  default:
    return 15 * time.Minute
}
```

[Erik Andersson]: This approach is dramatically cleaner than the current implementation:

> "So I think we can make this very, very clean compared to how it used to be."

**Function Call Sites**: Where `selectTimeout()` is called, the function must now receive the SQS message object to access `ApproximateReceiveCount`:

```go
// In message processing loop:
for _, message := range messages {
  timeout := selectTimeout(message)  // Pass message, not time
  // Use timeout for scheduling retry
}
```

**Maximum Retry Enforcement**: The code already contains logic to discard messages after a maximum number of retries. This existing mechanism must be preserved and validated to ensure messages are dropped after 75 retry attempts.

### Testing Strategy Using Mock CRM Service

[Erik Andersson]: The team has access to a **mock CRM service** that simulates CRM behavior without requiring a real CRM system.

**Mock Service Purpose**: 
- Fulfills the generic connector contract
- Can be configured to return any error response
- Supports testing without risking real customer systems
- Allows deterministic, repeatable test scenarios

**Mock Service Architecture**:

The mock service uses a URL-based installation approach. Instead of connecting to a real CRM, you point the integration service to the mock service URL and specify which CRM type it should simulate via the API key parameter:

```
Base URL: <mock-service-address>

Installation format:
  URL: <base-url>
  API Key: <CRM-ID>  // This parameter specifies which CRM to emulate
  
Examples:
  API Key: "edeal" → Mock responds as EDeal
  API Key: "fscu" → Mock responds as FSCu
  API Key: "maxo" → Mock responds as Maxo
```

**How to Identify a Mock Installation**:

Mock installations have characteristic attribute naming patterns. The mock service prefixes entity names to attributes:

Real CRM (E-Deal):
```
Entity: person
Attributes: first_name, last_name, email
```

Mock CRM:
```
Entity: person
Attributes: person_first_name, person_last_name, person_email
```

[Erik Andersson]: The attributes are always prefixed with the entity name, making mock installations immediately recognizable.

**Mock Service Capabilities**:

✓ **Supports**:
- Full sync with predictable, repeatable data
- Field mapping configuration
- Sync condition configuration
- Consent message handling
- All outbound events: email, SMS, form submissions, tool events
- Returns randomized UIDs as identifiers
- Generates consistent data on repeated calls (not random each time)

✗ **Does Not Support**:
- Real-time messages from the mock CRM back to Apsis One (could be implemented but hasn't been)

**Data Predictability in Full Sync**:

[Erik Andersson]: Full sync returns predictable data on each execution:

> "It will like here it will give you the same data back every time. So the data each if you have mapped say the first name will be Jesus. What is it? I think first name under score one and then first first name under score 2 etcetera."

The mock service generates values like:
```
first_name_1, first_name_2, first_name_3, ... first_name_10000
```

This allows building deterministic sync conditions and segments for testing.

**Deployment Approach**:

The mock service can be deployed to the staging environment without requiring integration into pull requests:

1. Make changes to the mock service code locally
2. Deploy manually to staging (outside the pull request process)
3. Keep the deployment separate from the actual backoff code changes
4. This allows testing the new backoff logic without code conflicts

### Testing the Backoff Change

**Test Scenario**: Simulate continuous CRM failures and verify message retry behavior

**Setup**:

1. Install the mock CRM service on staging environment
2. Configure mock service to return `500 Internal Server Error` for specific account/section combination
3. Modify the mock service response handler (likely in `server.go` or `echo_server.go`) to inject failures conditionally:

```go
// In POST handler for campaign events:
if request.account == "test-account" && request.section == "test-section" {
  return InternalServerError("Simulated CRM failure")
}
```

[Michal Rosikiewicz]: The conditional approach prevents affecting other test integrations:

> "And that that additional if will will prevent us from breaking every other mock integration, but just a little bit for us, yeah."

4. Send messages through the integration and observe retry behavior
5. Verify that:
   - Messages retry with the new 5-15 minute intervals
   - Messages are discarded after 75 attempts (~5 hours)
   - Queue clears faster than the 18-hour window
   - No messages are lost or stuck indefinitely

**Estimated Effort**: 2 story points for implementation and testing

---

## Queue Clearing and Message Dropping Mechanism

### Emergency Queue Clearing Strategy

When a customer misconfigures their system and sends an unusually large batch of events (e.g., 500,000 messages), it can block the entire outbound queue. The platform has a mechanism to selectively drop problematic messages without processing them.

[Erik Andersson]: Historically, this has been necessary:

> "In that case, like what we have had to do historically is to deploy a fix to to production where we essentially discard that whole sending. It's better that like those events are not sent rather than us not sending anything for any customer for like potentially days because that's not going to be a manageable situation."

### Implementation Location

**File**: Outbound worker queue message processor  
**Function**: `processMessage()` in the message processor engine

**Mechanism**:

The message processor has access to the **integration key (IK)** which contains:
- Account identifier
- Integration identifier  
- Section identifier

Before processing a message, the code can check if the message belongs to a problematic integration:

```go
// In processMessage function:
func processMessage(message *SQSMessage, ik *IntegrationKey) error {
  
  // Check if this message should be dropped
  if ik.Account == "problematic-account" && ik.Section == "problematic-section" {
    logger.Warn("Skipping message due to clogged queue", 
      zap.String("account", ik.Account),
      zap.String("section", ik.Section))
    return nil  // Return success without processing
  }
  
  // Normal message processing continues...
}
```

### Return Nil Semantics

When the message processor returns `nil` (no error):
1. The message is removed from the SQS queue
2. The system treats it as "successfully processed"
3. No call is made to the CRM system
4. The message is effectively discarded

This is extremely fast—thousands of messages can be cleared from the queue within minutes.

### Data Loss Mitigation

[Erik Andersson]: While this approach discards messages, the platform has safeguards:

> "We typically log everything. So we like we don't completely lose the messages or you can you could in theory send them like empty the dead letter queue and then send every message to the dead letter queue and redrive them, uh, later on, depending on how you want to, um, how you want to handle it."

**Options**:
1. **Logging**: All message details are logged before dropping, so they're not truly lost
2. **Dead Letter Queue (DLQ)**: Messages can be routed to a DLQ and reviewed/redriven later
3. **Approval Required**: This mechanism should only be used with consultation and approval due to data loss implications

### Queue Clearing Speed Benefit

[Michal Rosikiewicz]: This mechanism directly benefits from reducing the backoff delay:

> "So it's the second point for in favor for our 15 minutes mark Max time."

With the new 15-minute maximum delay:
- Readable messages can be cleared immediately (as soon as they leave the backoff phase)
- Queue can be drained in minutes instead of hours
- Doesn't require waiting for 6-hour or longer backoff periods

---

## CPU Alarm Tuning and False Positives

### Current False Alarm Problem

[Michal Rosikiewicz]: The team is receiving frequent CPU alarms for both the Sluice Worker and Delta Sync Manager services. These alarms are often false positives caused by transient CPU spikes rather than genuine capacity issues.

> "I'm bothered during this meeting the third time with high CPU in in what's the service in just in."

### Service Relationship

The two services are tightly coupled:
1. **Delta Sync Worker/Manager**: Receives burst of sync activity when customers change CRM metadata structure
2. **Sluice Worker**: Processes the output of Delta Sync, moving changes downstream

[Erik Andersson]: When the Delta Sync worker is overwhelmed, the Sluice worker experiences high load shortly afterward:

> "Those two services kind of go hand in hand because first the Delta sync is bombarded. If someone does a massive sync, even though we have said you shouldn't do it because they changed like their metadata structure or something and then every contact they have in the whole CRM system is updated and from the Delta Sync worker or this Delta Sync manager they are moved to the sluice worker."

### Current Alarm Configuration

**Sluice Worker**: 
- Configuration location: Sluice Worker SAM file (local configuration)
- Threshold: Relatively aggressive

**Delta Sync Manager**:
- Configuration location: Centrally managed in CloudFormation (manager_cloud_formation)
- Cannot be changed independently from other manager services
- Affects all manager functions equally

### Proposed Adjustment

[Michal Rosikiewicz]: Instead of triggering on a single CPU spike, add **evaluation periods** to require sustained high CPU:

> "It usually do like 5 consistent periods of like 60 seconds like so it it means that 5 minutes or or done it would be like sorry. Holding I need to know we need to look how the alarm is talking and adjust accordingly."

**New alarm logic**:
- Not: "Alert if CPU > X% for 1 minute"
- But: "Alert if CPU > X% for 5 consecutive 1-minute periods" (5 minutes sustained)

This filters out transient spikes from legitimate bursts while catching genuine sustained load issues.

### Tuning Approach

[Michal Rosikiewicz]: The team should review historical CPU graphs and adjust thresholds to match actual baseline behavior:

> "But we do it comparing the the graphs and historical data that we have. So if if this alarm was not the real alarm, then we should tweak this value. So it won't trigger an alarm next time."

**Steps**:
1. Examine CloudWatch CPU graphs for historical patterns
2. Identify what CPU levels are normal during regular operation
3. Identify what CPU levels truly indicate problems
4. Adjust alarm threshold and evaluation period accordingly
5. Iterate based on subsequent alarm history

### Scope of Change

[Erik Andersson]: Adjusting the evaluation period for all manager services (not just Delta Sync) is reasonable:

> "There would not be an issue to change this for all managers, I would say."

The issue of transient CPU spikes during normal operations is not unique to Delta Sync and would benefit all manager services.

---

## Architecture Notes and Gotchas

### SQS Message Metadata

SQS provides built-in metadata for messages that should be leveraged rather than reimplemented:
- `ApproximateReceiveCount`: Number of times the message has been retrieved from the queue
- Use this instead of calculating elapsed time or other heuristics

### Message Visibility and Backoff

Messages in the backoff state are "invisible" to the queue processing system:
- They won't be read or processed until the backoff period expires
- During backoff, you cannot immediately clear them from the queue
- This is why reducing backoff duration makes queue management more responsive

### Testing Limitations with Backoff

[Erik Andersson]: During development testing with the mock CRM, long backoff delays block progress:

> "But because we were putting like a four hour retry on this, we couldn't continue with the testing because I couldn't clear this message from the queue. I can't purge the queue in product for obvious reasons."

The production queue cannot be purged for safety reasons, but this highlights the importance of tuning backoff appropriately.

### Generic Connector Contract

The mock service "fulfills the generic connector contract," suggesting that all CRM integrations implement a common interface. This allows:
- Swapping real CRM implementations for mock implementations
- Testing customer configurations without touching live systems
- Simulating various CRM error conditions

---

## Key Takeaways

1. **Alarm Design**: The current alarm strategy fails when customers experience continuous errors—the alarm never resolves and operators lose visibility into the true failure volume. A dashboard-based approach is superior for persistent failure scenarios.

2. **Queue Risk**: The outbound worker's FIFO queue with a 20,000 message limit creates a cascading failure risk if one customer's messages block all others. The AVS message group search limitation means high-volume problematic messages can starve other customers.

3. **Backoff Redesign Is Critical**: The current 18-hour retry window is excessive. Reducing to 5 hours with proper tuning (5 seconds → 15 minutes plateau) is justifiable because if an issue isn't resolved within 4-5 hours, continued retries are unproductive. This also enables faster queue clearing in emergency scenarios.

4. **Implementation Is Clean**: Refactoring the backoff function to use SQS's `ApproximateReceiveCount` instead of calculating elapsed time reduces code complexity and makes it more maintainable. This is estimated at 2 story points.

5. **Mock Service Is Powerful**: The mock CRM service allows deterministic testing of retry logic and integration behavior without risk to production systems. It should be more widely known and used for testing.

6. **Alarming Requires Tuning**: CPU alarms need evaluation periods to filter transient spikes. The team should review historical data and adjust thresholds based on actual baseline behavior rather than theoretical maximums.

7. **Data Integrity Trade-offs**: Emergency message dropping may be necessary to prevent queue clogging, but should require approval. Messages can be logged or sent to DLQ before dropping to prevent true data loss.

---

## Unresolved Questions and Action Items

### Action Items

1. **Implement Backoff Function Refactor**:
   - Modify `lib/sqs.go` `selectTimeout()` function
   - Change to accept SQS message object instead of time
   - Implement switch-case on `ApproximateReceiveCount`
   - Update all call sites to pass message object
   - Verify maximum retry enforcement at 75 attempts
   - Estimated effort: 2 story points
   - Testing: Use mock service with conditional failure injection

2. **Set Up Mock Service for Testing**:
   - Configure mock CRM on staging environment
   - Modify mock service response handler to inject failures for specific account/section
   - Deploy separately from backoff code changes
   - Create test scenario for message retry behavior
   - Document mock service setup process for team

3. **Implement Monitoring Dashboard**:
   - Defer until Lukasz (missing team member) returns
   - Create AWS CloudWatch metrics for failed messages by account/section/integration
   - Determine data structure (statistics cache vs. database)
   - Scope dashboard functionality

4. **Tune CPU Alarms**:
   - Review historical CloudWatch CPU graphs for Sluice Worker and Delta Sync Manager
   - Determine appropriate threshold levels based on baseline behavior
   - Implement evaluation period requirement (suggest 5 consecutive 60-second periods)
   - Update CloudFormation for manager services
   - Update Sluice Worker SAM file
   - Monitor for false positives in next week

### Questions Requiring Clarification

1. **Message Aggregation Granularity**: Should the monitoring dashboard aggregate failures at account, section, or integration level? Or all three?

2. **Actionable Error Classification**: What criteria should be used to classify errors as "actionable now" vs. "wait for on-call review"? Should 401/permission errors go directly to SOC?

3. **Message Dropping Approval Process**: What is the formal approval workflow for using emergency message dropping? Who should be consulted?

4. **DLQ Strategy**: If messages are dropped to DLQ, what is the process for reviewing and redriving them?

5. **Mock Service Real-Time Messages**: Would implementing real-time message support in the mock service be valuable for testing customer-side integrations?