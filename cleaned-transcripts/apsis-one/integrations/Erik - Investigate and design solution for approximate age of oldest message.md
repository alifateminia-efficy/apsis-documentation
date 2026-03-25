---
source_file: Erik - Investigate and design solution for approximate age of oldest message.txt
domain: Apsis One Integrations
topics: [Outbound Worker Queue Management, Message Retry and Backoff Strategy, Error Monitoring and Alerting, Queue Congestion Handling, Mock Service Testing, CPU Alarm Configuration]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Outbound Worker, SQS Queue, Delta Sync Worker, Sluice Worker, Mock Service, Message Processor, CRM Integration]
session_type: architecture-review
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This knowledge transfer session focused on designing a solution for handling message age and queue congestion in the **Outbound Worker**, with particular emphasis on improving the retry and backoff mechanism. Erik Andersson walked through the current problems with continuous errors from customers, the 20,000-message queue limit threshold that can cause processing bottlenecks, and proposed refactoring the exponential backoff logic to be cleaner and more configurable. The discussion also covered using a mock CRM service for testing, clearing problematic messages from the queue when necessary, and adjusting CPU alarm thresholds.

---

## Page Duty Alarms and Error Handling Philosophy

### Current Alert Resolution Behavior

[Erik Andersson]: The design philosophy for page duty alarms is that when an actual error occurs in the system, the alarm triggers. If no further errors occur within a given time window, the alarm auto-resolves. If the same issue occurs again, it should trigger again—ideally after the on-call team has taken action to prevent recurrence.

### Where Errors Originate

**Manager Services**: Errors typically occur when customers perform operations like installations or full syncs, or when they input malformed data. These are often things the system did not anticipate.

**Outbound Worker**: Errors here are almost always connected to the CRM system. This is critical because the Apsis team cannot directly fix CRM issues—they must notify the CRM developers or SOC (for permission issues) to take action.

### The Problem with Current Approach

[Erik Andersson]: The current alarm system assumes errors are temporary and will auto-resolve. However, some customers have *continuous* errors that never auto-resolve. This means the alert stays in an active state indefinitely, hiding the fact that many messages are accumulating. The on-call team doesn't see the true volume of failures occurring.

---

## The 20,000-Message Queue Limit and Processing Bottleneck

### Queue Processing Limitation

[Erik Andersson]: ABS (the message processing system) only looks at the first 20,000 messages in the SQS queue when searching for message groups to process. If a customer has 20,000+ messages stuck in the queue for the same integration/section, any message behind those 20,000 will not be processed at all because ABS never reaches it.

### Why This Matters

If a customer misconfigures their system and sends, for example, 500,000 email events in one go, and those messages get stuck (timing out or receiving errors), they block the entire outbound queue for all other customers. This is a "nasty situation" that can persist for days if left unmaintained.

> "We might risk ending up in some nasty situation because of how AVS looks for the message groups to process."

### Current Workaround

Historically, the team has had to deploy a production fix to discard stuck messages from problematic customers to unblock the queue for everyone else. This is preferable to having all message processing halted for all customers.

---

## Monitoring and Error Visibility Strategy

### Current Approach: Post-Incident Analysis

[Michal Rosikiewicz]: After each on-call week, the team runs weekly queries in log insights to verify what happened. Based on results, they either:
- Plan fixes for issues the team caused
- Notify the customer that something is failing on their end and they need to fix it

### Proposed Approach: Dashboard with Actionable Metrics

[Erik Andersson]: Rather than relying solely on alarms (which don't auto-resolve for continuous errors), the team should build a dashboard that tracks:
- Failed messages per account/section/integration
- Permission errors (401s from rotated API keys)
- Retrieval errors anywhere in the system

This dashboard would allow on-call staff to check it at the start of their shift and see what happened overnight without waiting for alarm notifications. For some error types (like permission issues), the system could even automatically route them to SOC.

[Michal Rosikiewicz]: The team is open to incorporating existing log insights queries into the integration system, which would be a step toward this monitoring improvement.

---

## Retry and Backoff Mechanism Refactoring

### Current Implementation Issues

[Erik Andersson]: The current retry logic uses a function that calculates the backoff time based on **how long the message has been in the queue**, rather than using SQS's built-in metadata about **how many times the message has been retried**.

```
Current approach: Compare total elapsed time against thresholds
- Sets retry multiplier based on time-in-queue
- Maximum of 8 retries with exponential backoff
- Total retry window: approximately 1.5 days
```

[Erik Andersson]: This function is "really, really ugly" and should be refactored to use SQS's native `ApproximateReceiveCount` metadata instead.

### The Problem with Current Backoff Sequence

Current retry intervals: 15 seconds → 15 seconds → 1 minute → 5 minutes → 15 minutes → 1 hour → 4 hours → 6 hours, then discard after 5 hours (contradiction in implementation).

**Issue**: If a CRM is down for 4 hours and comes back up at hour 4.5, the message has already been retried 7+ times and may be about to be discarded. Conversely, if a CRM has a genuine bug that takes the team 8 hours to fix, the message is already discarded after ~5 hours.

[Michal Rosikiewicz]: If someone cannot fix an issue within 4 hours, the message shouldn't be retried for another 1.5 days.

### Proposed New Backoff Strategy

[Erik Andersson]: Implement a simple switch-case approach based on retry count:

```go
switch retryCount {
  case 0:
    return 15 seconds
  case 1:
    return 30 seconds
  case 2:
    return 1 minute
  case 3:
    return 5 minutes
  case 4:
    return 15 minutes
  default:
    return 15 minutes  // Continue retrying every 15 minutes
}
```

**Total retry window**: 5 hours maximum (300 minutes / 15 minutes = 20 retries max, but implementation adjustable)

[Erik Andersson]: "It's so much more prettier" than the current implementation.

### Why This Is Better

1. **Cleaner code**: Uses a straightforward switch statement instead of calculating elapsed time
2. **SQS metadata**: Leverages the built-in `ApproximateReceiveCount` field in SQS message attributes
3. **Faster incident response**: Reduces total retry time from 1.5 days to ~5 hours, allowing faster queue clearing if needed
4. **Easier tweaking**: Adjusting retry intervals only requires changes to the function signature and switch cases

### Implementation Details

The function signature must change to accept the SQS message as input:

```go
func selectTimeout(sqsMessage *sqs.Message) time.Duration {
  retryCount := sqsMessage.Attributes["ApproximateReceiveCount"]
  // switch statement here
}
```

Where the function is called (in the message iteration loop), the code already has access to the full SQS message metadata, so passing it in is straightforward.

### Related Refactor: The Ugly Function

[Erik Andersson]: The current function calculates backoff by comparing elapsed time. The metadata field name for retry count in SQS is something like `ApproximateNumberOfReceiveAttempts` or similar (Erik had to search for the exact field name during the session).

---

## Where Retry Backoff Is Applied

The refactored backoff function needs to work in two places:

1. **Sluice Worker**: Applies backoff when a full sync is running and messages are being retried
2. **Outbound Worker**: Applies backoff when CRM system errors occur

[Erik Andersson]: These are the two places where backoffs are frequently applied. The function only needs to be changed in one place, and both services will benefit.

---

## Testing the Backoff Changes with the Mock Service

### What Is the Mock Service?

[Erik Andersson]: The mock service is a customizable fake CRM environment that fulfills the generic connector contract. It allows testing against a CRM you control without breaking actual customer systems.

**Key endpoint**: The mock service runs at a specific URL that can be configured in an integration installation. By specifying an API key that identifies the CRM type, you can make it behave as different CRM systems (E-deal, Efficy CRM, iMartic/Maxo, etc.).

### How to Set Up a Mock Installation

In the Integrations UI:
1. Provide the mock service URL as the integration address
2. Use the API key field to specify which CRM type to simulate:
   - `E-deal` for E-deal
   - `FECU` for Maxo (iMartic)
   - etc.

**Result**: The mock installation will return the appropriate entity names and field structures for that CRM type.

### Mock Service Behavior

- **Full sync**: Works and returns predictable data (e.g., `firstname_1`, `firstname_2`, etc.) so test conditions can be created
- **Create/Update operations**: Returns success and a random UID as the identifier
- **Attributes**: Prefixes attribute names with the entity name (e.g., `person_email` for E-deal) making it easy to spot it's a mock
- **Limitation**: Does not support inbound real-time messages from the CRM back to Apsis (could be implemented but hasn't been)

### Testing the Backoff Logic

To test that messages are retried and discarded after the new backoff window:

1. Set up a mock installation for testing
2. Configure the mock service to return a 500 error for specific account/section combinations:
   ```go
   // In the post campaign events function (server.go in mock service)
   if params.account == "test_account" && section == "test_section" {
     return 500 InternalServerError
   }
   ```
3. Send test messages to that installation
4. Verify backoff behavior: messages retry at configured intervals and are discarded after 5 hours

**Important**: Use conditional logic to only return errors for your test account/section, not all integrations, to avoid breaking other teams' tests.

### Deployment of Mock Changes

[Erik Andersson]: Mock service changes can be deployed separately from the integration code changes. You can:
- Push mock changes manually without including them in your pull request
- Deploy the mock to staging
- Test the integration code against the updated mock
- Then merge the integration code separately

This avoids holding up the integration PR on mock service deployment.

---

## Creating the Story for Backoff Refactoring

The task agreed upon during the session:

**Title**: "Improve exponential backoff retry logic in outbound worker using SQS message metadata"

**Description**: 
- Change `selectTimeout` function to accept SQS message metadata instead of calculating elapsed time
- Use `ApproximateReceiveCount` from SQS message attributes to determine retry interval
- Replace the current 8-retry / 1.5-day window with a simpler approach: retry every 15 minutes for a maximum of 5 hours total
- Implement as a clean switch-case statement based on retry count
- Update both Sluice Worker and Outbound Worker where applicable
- Test with mock CRM service returning 500 errors to verify messages are discarded after 5 hours

**Effort estimate**: 2 points (simple change with testing)

**Testing approach**: Use the mock service (see section above) configured to return errors for a test account/section, send messages, and verify they are retried at the new intervals and discarded after ~5 hours.

---

## Message Deletion from Queue for Clogged Queues

### When and Why

[Erik Andersson]: If a customer misconfigures their system and sends massive amounts of events (e.g., 500,000 at once), and those messages get stuck, it blocks processing for all other customers. In such cases, historically the team has deployed a fix to production to delete those messages.

> "It's better that like those events are not sent rather than us not sending anything for any customer for like potentially days."

### How to Delete Messages

In the **Outbound Worker Message Processor** (`message_processor.go` or equivalent), in the `processMessage` function:

```go
if integrationKey.Account == "problem_account" && 
   integrationKey.Section == problem_section_id {
  logger.Warn("Skipping message due to clogged queue", 
    "integrationKey", integrationKey)
  return nil  // Return success without processing = message removed from queue
}
```

**Why this works**: Returning `nil` (no error) tells the queue processor that the message was successfully processed, so it's removed from the queue. No CRM API call is made.

**Speed**: Messages can be cleared at a rate of tens of thousands per minute using this approach.

### Data Loss Considerations

[Erik Andersson]: While this approach does result in messages not being sent to the CRM, it's still preferable to blocking all customers. The team logs everything, so messages can theoretically be recovered from logs later.

Alternative approach: Send all discarded messages to a dead letter queue and manually redrive them later.

### Requires Approval

[Erik Andersson]: This should not be done without consulting and getting approval from leadership, but in emergency situations where queues are clogged, it's the right move.

### Connection to 15-Minute Backoff

[Michal Rosikiewicz]: With the 15-minute max backoff interval, the queue can be cleared much faster—within minutes rather than waiting 6+ more hours for the current exponential backoff to complete.

---

## CPU Alarm Threshold Issues in Managers

### Problem

[Michal Rosikiewicz]: The team has been receiving CPU alarms multiple times during this call, specifically for the Delta Sync Worker. The alarm threshold is too sensitive and triggers on normal spikes in activity.

### Current Behavior

- **Delta Sync Manager** and **Sluice Worker**: Both experience CPU spikes when customers perform large syncs or metadata changes
- When someone changes metadata structure in the CRM, every contact is marked as updated
- These updates flow from Delta Sync Manager → Sluice Worker
- Current alarm: Triggers immediately or after very short evaluation period

### Proposed Change

[Michal Rosikiewicz]: Extend evaluation period to require CPU to stay high for multiple consecutive periods before alarming.

**Suggested approach**: 5 consecutive evaluation periods of 60 seconds each = 5 minutes of sustained high CPU before alarm triggers.

Alternatively: Change to something like "CPU > 94% for more than an hour" to distinguish between normal spikes and genuine problems.

[Tomasz Kowalski]: The alarm configuration should be compared against historical graphs to determine if it's actually a false positive, then adjusted accordingly.

### Configuration Location

- **Sluice Worker**: Configured within the Sluice Worker SAM template
- **Delta Sync Manager**: Configured in the centralized Manager CloudFormation template, so changes apply to all managers

[Erik Andersson]: Changing the Delta Sync Manager alarm would not negatively affect other managers, so making it more permissive is reasonable.

---

## Handling Continuous Errors vs. Temporary Errors

### The Core Design Tension

[Erik Andersson]: The alarm system assumes errors are temporary events that resolve themselves. But continuous errors from misconfigured or problematic customers violate this assumption:
- Alarm triggers
- Errors keep occurring
- Alarm never resolves (because errors never stop)
- On-call person doesn't see an overview of the volume of failures

### Solutions Discussed

1. **Dashboard approach** (preferred): Track failures per account/section/integration and display on a dashboard the on-call team checks at start of shift
2. **Warnings instead of alarms**: Convert some continuous errors to warnings, but this requires manual review
3. **Dedicated monitoring service**: Similar to how other services handle it, with a dedicated DevOps/monitoring team
4. **Hybrid**: Keep alarms for critical issues, use dashboard for detailed error tracking and prioritization

[Michal Rosikiewicz]: The current approach is weekly post-analysis using log insights queries. A dashboard would be an improvement but requires process redesign since that's not how integration currently works.

---

## Architecture and Error Types

### Permission Errors and API Key Errors

[Erik Andersson]: Certain errors are immediately actionable and should be routed to SOC:
- 401 Unauthorized: API key has been rotated or is invalid
- 403 Forbidden: Permission issue in the CRM instance
- SQL errors inside the CRM database (requires CRM developers to fix)
- HTML errors returned by CRM (indicates server-level issues)

### Temporary vs. Permanent Errors

**Temporary** (retry is appropriate):
- CRM temporarily down for maintenance
- Network blip
- Transient database lock

**Permanent** (retry won't help; requires action):
- Missing required fields in CRM database (fixed by CRM team)
- API key rotated (fixed by SOC)
- Malformed data sent by customer (fixed by customer)
- CRM software bug (fixed by CRM developers)

### Example from Real Incident

[Erik Andersson]: During this morning's testing, the CRM team failed to complete a database migration and were missing required fields in their test environment. This error was visible in logs immediately, but because the system was configured with a 4-hour retry interval, the test couldn't proceed—the message couldn't be cleared from the queue without purging production (which is not allowed).

> "We saw this error here in the logs, so we knew exactly what was the problem. He fixed this in real time, but because we were putting like a four hour retry on this, we couldn't continue with the testing because I couldn't clear this message from the queue."

Reducing the retry interval to 15 minutes would have allowed faster testing and iteration.

---

## Key Takeaways

1. **Backoff refactoring is high-value**: The new switch-case approach based on retry count is simpler, more testable, and allows faster queue clearing (5 hours vs. 1.5 days)

2. **Mock service is underutilized**: The team has a powerful testing tool (mock CRM) that can be used to test retry behavior, error handling, and queue dynamics without risking production or customer systems

3. **Queue congestion is a real problem**: The 20,000-message limit means customer-specific issues can block all other customers; the team needs both prevention and clearing mechanisms

4. **Monitoring needs improvement**: Current alarm-based approach doesn't surface continuous errors well; a dashboard showing failures per account/section would help on-call staff prioritize and take action faster

5. **CPU alarms are too sensitive**: The Delta Sync Manager and other managers generate legitimate CPU spikes that shouldn't trigger alerts; evaluation periods should be extended

6. **Error routing needs process redesign**: Some errors (permission, API key, CRM-side bugs) should be routed to SOC or CRM team immediately; this requires process and possibly architectural changes beyond just the code fix

---

## Unresolved Questions and Action Items

### Agreed-Upon Tasks

1. **Refactor backoff function** (2 points)
   - Change `selectTimeout` to use SQS `ApproximateReceiveCount`
   - Implement new 15-minute interval strategy with 5-hour max
   - Test with mock service returning 500 errors
   - Verify in both Sluice Worker and Outbound Worker

2. **Update mock service for testing** (part of backoff task)
   - Modify mock service echo server to return 500 errors for specific account/section combinations
   - Deploy mock service separately to staging for testing
   - Include instructions for test teams on how to use it

3. **Adjust CPU alarm thresholds** (pending)
   - Review historical CPU data for Delta Sync Manager and Sluice Worker
   - Determine appropriate threshold and evaluation period (e.g., 5 consecutive periods or 1-hour sustained high CPU)
   - Update CloudFormation template for managers
   - Update SAM template for Sluice Worker

### Deferred/Pending Discussion

1. **Dashboard implementation for error monitoring**: Deferred pending Lukasz's availability; requires process discussion alongside technical implementation

2. **Message deletion automation**: Policy discussion needed—when should automatic message dropping be triggered vs. requiring manual approval?

3. **Error routing to SOC**: Architectural change needed to automatically route permission/API key errors to SOC rather than on-call team

4. **Retry strategy specification**: Team needs to formally document and agree on retry intervals and maximum retry time in specifications

---

## Technical Details Preserved for Reference

**SQS Message Metadata Field**: `ApproximateReceiveCount` (or similar; exact name should be verified in AWS SQS documentation)

**Mock Service Configuration**: Accepts API key parameter to specify CRM type; returns appropriately structured entities and attributes for each type

**File Paths Discussed**:
- Message processor: `message_processor.go` (Outbound Worker)
- Mock service: `echo_server.go` (mock service repository)
- Sluice Worker SAM: CloudFormation SAM template for Sluice Worker
- Manager CloudFormation: Centralized template managing all manager functions (Delta Sync, etc.)

**Functions to Change**: `selectTimeout()` function signature and implementation in the backoff/retry system

**Current Retry Sequence**: 15s, 15s, 1m, 5m, 15m, 1h, 4h, 6h (8 retries, ~1.5 days total)

**Proposed Retry Sequence**: 15s, 30s, 1m, 5m, 15m recurring (up to 75 attempts or 5 hours, whichever comes first)