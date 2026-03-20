---
source_file: Erik - Incidents in Integrations.txt
domain: Apsis One - Integrations
topics: [Incident Management, SQS FIFO Queue Limitations, Database Migrations, Error Handling, CRM System Integration, Installation/Uninstallation Workflows, Message Queue Monitoring, HTTP Error Responses, Backoff Strategies]
speakers: [Erik Andersson (Integration Lead), Michal Rosikiewicz, Tomasz Kowalski, Lukasz Grabowski (Transcriber)]
key_components: [Integration Manager, Mappings Manager, Full Sync Service, Delta Sync Queue, Outbound Worker, SQS FIFO Queues, Integration Database, CRM Systems (Enterprise, Dynamics, E-Deal, Tribe, Web CRM), AWS Lambda]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson led a comprehensive knowledge transfer session on incident management and operational concerns within the Apsis One Integrations platform. The session covered three critical incident categories: SQS FIFO queue message grouping limitations that can block processing across customers, database migration requirements that must be manually executed before releases, and a severe historical incident in the MA (Marketing Automation) service involving stuck HTTP requests. The session also addressed common customer-facing issues during installation/uninstallation workflows and CRM system compatibility challenges. Throughout, Erik emphasized recovery mechanisms available (full sync, data restoration) and provided practical guidance for distinguishing error severity and determining which logs to consult for different failure scenarios.

---

## Data Resilience and Recovery Capabilities

### Nature of Data Loss in Integration Services

Erik established a critical framework for understanding data loss risk in integration contexts:

> By nature, integration is not a very incident-prone service. The most dangerous case of an incident is customer being impaired, but data loss is like the most would be the worst case. However, by nature it is quite hard to encounter real data loss in integration.

The distinction between **failed message processing** (a failed message delivery) and **irreversible data loss** (inability to restore customer data to a proper state) is fundamental. Failed message processing alone does not constitute data loss because the source data persists in either the CRM system or Apsis.

### Recovery Mechanisms: Full Sync as Restoration Tool

The **full sync** process is the primary recovery tool and works bidirectionally:

- **Download phase**: Full sync downloads every contact and all mapped data fields, plus all consent records, from the CRM system into Apsis
- **Upload phase**: Full sync exports consent state from Apsis back to the CRM when the CRM's consent state lags behind Apsis
- **Outage recovery**: Can recover from network issues, bugs in Apsis, bugs in the CRM, and incorrect delta sync configurations

Erik noted that even if an incorrect delta sync manager was deployed (discarding real-time syncs), the underlying data remains intact in the CRM, making restoration possible. However, full sync is resource-intensive on customer CRM systems, so it should not be run too frequently. Internally triggered full syncs are now possible but require careful consideration of CRM load.

---

## SQS FIFO Queue Limitations and Message Grouping

### The AWS SQS Message Group Scanning Limitation

The most significant operational constraint affecting Integration is an AWS implementation detail: **SQS FIFO queues only scan the first 20,000 messages for message group identifiers**. This limitation creates a critical bottleneck when processing large bursts of messages.

#### How the Limitation Manifests

**Original message grouping strategy** (now changed):
```
Message Group ID = account_section + integration_id
```

When a customer triggered a major metadata change in their CRM (e.g., adding a new field to all contacts), it could generate 200,000+ contact update messages in rapid succession. If a backoff was placed on the message group due to a failure partway through processing:

1. SQS would only check the first 20,000 messages for that message group
2. If the backoff was on message 1-120,000, SQS would find it (within the first 20,000)
3. However, all subsequent messages from ANY other customer would queue behind these 120,000 stuck messages
4. No processing would occur for any other customer until the queue was drained below 20,000 messages
5. No errors or logs were generated, so monitoring systems didn't alert

> What we had to do in these cases was essentially to put a blacklist on this specific installation, discard all of those messages that were in the queue for that customer. In the consumer, we just said if the message comes from this installation, like just throw it away. And then we informed SOC saying like, yeah, we had to do this because they flooded our system. And the customer needs to do a full sync.

#### Current Message Grouping Strategy (Mitigation)

The grouping strategy now includes the CRM ID:
```
Message Group ID = account_section + integration_id + crm_id
```

This ensures that a backoff only affects messages for a specific contact from a specific customer, not all contacts from that customer, allowing other customer messages to be processed independently.

### Two High-Risk Queue Scenarios

#### 1. Delta Sync Queue

**Purpose**: Receives real-time webhook messages when customer data changes in their CRM

**Risk trigger**: Customer performs a bulk data model change without disabling webhooks first (e.g., adding a field to all 100,000+ contacts)

**Impact**: Can receive massive burst of messages if customer does not follow the recommended practice of temporarily disabling webhooks during database migrations

**Mitigation**: The revised message grouping (including CRM ID) significantly reduces this risk compared to the original design

#### 2. Outbound Worker Queue

**Purpose**: Processes outbound events from Apsis (sent, delivered, opened, clicked, viewed)

**Risk trigger**: E-mail activity configuration mapped for syncing when customer has large recipient bases

**Impact**: A single email send to 100,000+ recipients can generate 500,000+ events in the queue (multiple event types × number of recipients)

**Real example**: Hula Press in Tribe can put approximately 500,000 events into this queue

**Current handling**: Events are grouped/batched, so the queue generally handles this well, but it remains an area requiring monitoring

### Queue Monitoring Alerts

**Alert threshold**: Messages remaining in queue for **over approximately 1 hour**

The alarm is based on **message age**, not message count, because:
- Large jobs (e.g., bulk exports) can legitimately trigger 200,000+ messages without being problems
- The issue is when messages stop being processed, indicated by old messages remaining in queue

**What to check when alarms fire**:
- How many messages are currently in the queue?
- Are there errors in the consumer logs?
- Is there a backoff active? If so, on which message group?
- Is the backoff preventing any other customer's messages from being processed?

---

## Database Migrations and Release Requirements

### Migration Architecture

Integration uses a dual-file migration pattern:

**Schema Create File** (`create_schema.sql`):
- Complete database structure for entire Integration service
- Used only when setting up database completely from scratch
- Not used for incremental updates

**Delta Files** (`delta_*.sql`):
- Iterative SQL updates applied when features add new tables or modify schema
- Applied one at a time before each release
- Must be applied manually—no automatic migration system exists

### Critical Pre-Release Checklist

Before merging code to `product` or `master` branches:

1. **Check for delta file changes**: Look at file changes in the merge request
2. **If delta files changed**: Manually apply them by establishing a database connection and running the corresponding SQL files
3. **Timing**: Must be done BEFORE the release, not after

Erik acknowledged that no automatic migration mechanism currently exists in Golang for this pattern:

> The migration in integration is not automatic. We have here a delta SQL file... but we've failed so far to find any nice automatic way to do this in Golang, but it might be possible like doing doing it with some custom bash script or if there has been any new framework released to handle this.

### Impact of Missed Migrations

**Severity**: Moderate to High (alerts quickly but recovery is annoying)

When a service encounters a missing column or table:
- The service fails with a clear error logged in paid duty
- The failure is rapid and obvious (not silent)
- Recovery requires customer to run a full sync
- Real-time sync messages are retried, so they process successfully once migration is applied

The issue is primarily operational inconvenience rather than data loss, but it can impact customers. Erik strongly encouraged finding a way to automate this process.

---

## Historical Severe Incident: MA Service HTTP Client Deadlock

### The Incident Overview

The most serious incident the Integration team has handled involved the MA (Marketing Automation) service, though it also potentially affected Integration. The incident resulted in 400,000+ jobs stuck in the queue for over 2.5 weeks before detection.

### Root Cause Chain

1. **Audience CRM change**: Audience modified how they handle HTTP 429 (rate limit) responses—they removed the response body and now return only the status code `429`

2. **Shared HTTP client library defect**: A shared library used in MA had a bug: if an HTTP response was non-successful and the request body was empty, the response was never properly terminated/closed

3. **Consumer deadlock**: The MA service runs multiple consumer workers that process messages from queues. When a consumer encountered this non-terminating HTTP response:
   - The request hung indefinitely
   - The function never returned
   - No success log was generated
   - No error log was generated
   - The consumer became stuck on that single message

4. **Silent failure**: Because no logs were produced, monitoring systems (paid duty) never triggered alarms

5. **Detection lag**: The incident went unnoticed for **2.5 weeks** until someone manually checked and found 400,000+ jobs stuck

### Why MA Service Impact Was Severe

Unlike Integration, MA (Marketing Automation) is **time-sensitive**:

> MA in contrast to integration is very time sensitive because let's say that the customer has set up a flow where like if this event occurs on a profile then send out this discount offer to the customer. And if that message were to be processed like 2 1/2 week later and not within the one hour limit that is imposed, like the customer will have a lot of very, very angry customers.

Delayed processing could result in:
- Missed purchase windows for discount offers
- Broken confirmation emails sent weeks late
- Broken customer experience and trust

**Recovery couldn't be automated**: Unlike Integration, you cannot simply replay these jobs. The team had to:
- Contact each affected customer individually
- Ask which flows should be retried
- Manually approve each retry
- Sit on this cleanup for 1.5 weeks (Erik and Shervidia working evening hours)

### Mitigation: Job Age Monitoring Lambda

Post-incident, a **Lambda function was implemented to continuously monitor job age**:

```
Lambda monitors: how old is the oldest message in the queue?
Alert trigger: when messages exceed acceptable age threshold
```

This approach avoids false positives from legitimate large bulk jobs while catching genuine stuck jobs.

**Why not just count messages?** When a customer triggers a bulk export, they might intentionally create 200,000 messages, which would be a false positive alarm if based on count alone.

---

## Installation and Uninstallation Error Handling

### Common Error Patterns

Most common customer-facing issues occur during installation and uninstallation workflows, which are handled by the **Integration Manager** service.

#### Explicit Error Messages (Good Cases)

In some scenarios, Integration provides clear, actionable error messages:

- **Invalid URL during installation**: "This URL is a bogus URL, we can't find it. You entered an incorrect URL."
- **Authentication failure**: "These credentials are not working. We received a 401 response."

#### Ambiguous Errors (Common Cases)

For most other errors, Integration returns:

> "Unknown error occurred. Here is the integration error ID: [ID]"

This occurs because:
1. Integration may receive unexpected errors from the CRM system
2. Integration doesn't know the root cause
3. CRM developers must investigate the error server-side

**Common uninstall error**: When Integration tries to delete webhooks during uninstallation, the CRM returns an unexpected error. Integration refuses to complete uninstallation because it wants to guarantee all trailing webhooks are deleted (so they don't continue sending data to Apsis). The response to the customer is the ambiguous "unknown error" message.

### Troubleshooting: Log Selection Guide

When customers report installation or uninstallation failures, the **service name in the error description points to which logs to check**:

| Customer Issue | Service Logs to Check | Notes |
|---|---|---|
| Failed during installation | Integration Manager logs | Handles all installation flow |
| Failed to save mappings | Mappings Manager logs | Dedicated mappings service |
| Full sync failed | Full Sync service/job logs | Check both producer and consumer |
| Failed during uninstallation | Integration Manager logs | Handles all uninstallation flow |
| Issue creating/syncing activity | Service matching activity type (e.g., Email, Form) | Check outbound flow logs |

**General approach**: SOC support tickets usually contain thorough descriptions. From the description, determine whether the problem is in the inbound flow, outbound flow, or setup phase. This eliminates ~90% of possible logs to search through.

> Based on the description of the problem, depending on that description like you would know is the problem in the inbound flow, is the problem in the outbound flow or is the problem like trying to set up the integration so like you can exclude 90% of the logs just from the error description.

### Manual Integration Deletion

**Scenario**: Customer failed to uninstall an integration and needs it completely removed

**Process**: Call the Integration Manager clear integration endpoint with specific flags

**Recommended approach**: Use the Postman collection provided for the operation rather than raw curl

---

## CRM-Specific Errors and External System Failures

### Classification of "Unknown Error" Responses

When Integration receives a non-success HTTP response from a CRM system during critical operations (campaign creation, webhook registration, consent updates), it returns an "unknown error" to the customer. These errors require **CRM system investigation**, not Apsis code changes.

**Common scenarios**:
- CRM returns 500 (internal server error) when creating a campaign
- CRM returns 400 (bad request) during webhook registration with an obscure message
- CRM returns 400 with a validation error that doesn't match Apsis's understanding

### CRM Developer Collaboration Channels

For each CRM system, there is a dedicated Teams channel in the "FCCRM and Marketing Connect Connector" group:

- **E-Deal**: Dedicated channel
- **Enterprise**: Dedicated channel (email-preferred; they dislike Teams)
- **Tribe**: Dedicated channel
- **Web CRM**: Dedicated channel

**Enterprise special handling**: The Enterprise team refuses to use Teams for issue reporting. All customer-specific problems must be reported via email, not Teams chat. Strategic discussions (like "how would this feature affect Enterprise?") can happen in Teams, but support issues must go to email.

### Example: Campaign Name Length Restriction

One example Erik encountered:
- A customer named an email campaign in Apsis with a very long name
- Enterprise had imposed a character limit on campaign names in their system
- Enterprise refused all requests to create campaigns with names exceeding their limit
- The error was not obvious from the response body
- Required investigation by Enterprise developers to identify the constraint

---

## Error Severity Classification and Pager Duty Tuning

### The Problem with Treating CRM Errors as Critical Alarms

Many errors that currently trigger pager duty alerts cannot be resolved by Apsis engineers at 3:00 AM because they require CRM system reconfiguration or investigation.

**Criteria for actionable errors** (should trigger alerts):
- Can be fixed by deploying an Apsis code patch
- Can be fixed by an Apsis engineer right now
- Indicates a bug in Apsis logic

**Criteria for non-actionable errors** (should be warnings instead):
- Require CRM system reconfiguration or investigation
- Indicate a problem on the CRM side that needs their team
- Cannot be resolved by any Apsis code change

### The 400 Status Code Ambiguity Problem

HTTP 400 (Bad Request) can indicate two very different problems:

1. **Apsis bug**: Apsis generated an invalid request payload
   - **Action**: Deploy a fix to Apsis code
   - **Severity**: Critical
   - **Should alert**: Yes

2. **CRM constraint**: Apsis sent a valid-looking request, but CRM has a configuration or constraint that rejects it
   - **Action**: Contact CRM team for investigation
   - **Severity**: Customer-blocking but not actionable by Apsis
   - **Should alert**: No (or as warning)

> The combination here of like the 400 and in particular this error message that is not something I can handle, so more often than not. You cannot just say OK, the status or the HTTP error was 400. That means I don't treat this as an error. It's the combination of the status and the error message usually, which makes me say...

### Proposed Tuning Strategy

Erik proposed a judgment-based approach: examine both the HTTP status code AND the error message to determine if the error is actionable by Apsis engineers. If the combination clearly indicates a CRM-side problem that cannot be fixed by deploying code:

- **Downgrade from error to warning** in the alerting system
- Customer experience is still preserved (the warning gets logged)
- Apsis teams are not woken up at night for non-actionable issues
- CRM teams still get notified via email/Teams

**Context-dependent**: Office-hours-only pager duty is less urgent, so some non-actionable errors can remain as alerts (someone will see them quickly and email the CRM team). But for 24/7 on-call schedules, downgrading non-actionable errors to warnings is essential to avoid alert fatigue and unnecessary pages.

---

## Recent Bug Fix and Testing: Dynamic Connector Configuration

### The Bug: Missing Configuration for Dynamics Connector Events

**Affected feature**: Consent timeline event syncing (recently released)

**What was broken**: When a new Dynamics connector installation was created, it failed during consent updates because the consent event didn't exist on the profile.

**Root cause**: The feature added event definitions to `generic_spec.go` for all connectors, including Dynamics. However, the configuration file that bootstraps connectors on startup was not updated to load this generic spec for Dynamics.

**Configuration file location**: `lib_config_connectors.go`

This file defines the inventory of available connectors and specifies:
- Which custom attributes to bootstrap on installation
- Which custom events to bootstrap on installation

**The fix**: Add Dynamics to the configuration so that when Dynamics is installed, the consent timeline events from `generic_spec.go` are loaded and bootstrapped.

### Discovery and Release Process

Erik discovered the bug during evening testing on beta after the initial release. He found that consent updates failed because:
1. The consent flow tries to add a consent event to the profile
2. The event didn't exist (because it wasn't bootstrapped)
3. Every consent update failed

The fix was:
1. Merged to beta branch (already reviewed by team)
2. Merged to master (production release in preparation)

**Lesson**: This type of bug highlights the importance of end-to-end testing with actual feature workflows, not just unit tests.

---

## Proposed Future Work: Integration Statistics Service

### Business Need

Product teams repeatedly request information that currently requires manual database queries:
- How many customers have installed a specific integration?
- How many customers are actively using a specific integration?
- What is the complete list of accounts/sections where a specific integration is installed?

Currently, the only way to answer these is to:
1. Access the Integration database
2. Query the `installation` table
3. Export data and manually analyze for permutations

### Proposed Solution: Simple Statistics Endpoint

A lightweight new service that would provide:
- REST endpoint: "Give me all integrations" 
- Returns: List of accounts, sections, and integration details
- Consumer: Back Office UI (new statistics tab)
- Implementation scope: Very basic but high value

### Why This is a Good Learning Project

Erik proposed this as the next collaborative feature to build, with the following benefits:

1. **Minimal implementation**: Can be finished in 2-3 days with proper walkthrough (half a day for Erik alone)
2. **Touches every required component**:
   - Design OpenAPI specification first (forcing documentation discipline)
   - Generate server interfaces from the spec
   - Implement the endpoint in Integration Manager service
   - Integrate with Back Office frontend
3. **Introduces OpenAPI-first development approach**: The team now defines endpoints in OpenAPI specification before cloud formation, ensuring documentation is always current
4. **Avoids duplication**: Previous approach required defining endpoints both in CloudFormation AND Swagger, causing sync issues

### Design Approach: OpenAPI-First

The current development workflow:
1. Define endpoint in OpenAPI specification
2. Generate server interfaces from the spec
3. Implement the interface functions
4. Push updated API docs automatically

**Benefits**:
- Single source of truth for API contracts
- CloudFormation files are cleaner (no duplication)
- Always up-to-date internal documentation
- Forces consideration of API design before implementation

---

## SQS Monitoring Deep Dive: Message Age vs. Count

### AWS SQS Limitations in Visibility

One annoying characteristic of SQS: you cannot query the queue for messages matching a specific message group ID. When investigating old messages in a queue:

**What you CAN see**: 
- The age of the oldest message in the queue (from CloudWatch metrics)
- Total message count (misleading without context)

**What you CANNOT easily see**:
- Which message group(s) are stuck
- How many messages are in a specific group
- Which customer's installation is blocking the queue

### Exponential Backoff Consequences

When an error occurs on a message, SQS applies exponential backoff before retrying:

**Backoff sequence**: 
- 1st retry: ~2 seconds
- 2nd retry: ~1 minute
- 3rd retry: ~5 minutes
- 4th retry: ~15 minutes
- 5th retry: ~1 hour
- 6th retry: ~3 hours
- 7th retry: ~5-6 hours
- Then messages are discarded or moved to dead letter queue

**The cascading problem**: If a CRM system is malfunctioning and rejecting all messages from a specific customer:
1. The first message in queue gets the longest backoff (e.g., 12 hours)
2. All subsequent messages from that customer queue behind it
3. The second message experiences the same CRM error, gets its own 12-hour backoff
4. By message 998 in the queue, you have a cumulative backoff of days

> If you have a malfunctioning CRM system and you have like 1000 messages for this and every message failed and like for the first message in the queue it like you will have a back off of like say 12 hours and then eventually like this will just be thrown away. Or it will be put in the dead letter queue, then the next message will be processed. But if the problem still is there...then you again will have like a 12 hour delay for that message.

This is one reason why identifying stuck message groups quickly is critical—the cost of delayed resolution increases exponentially.

### Investigating Old Queue Messages

When investigating an alarm about old messages in queue:

1. Check the **outbound worker logs** (for Outbound Worker queue) or **Delta Sync logs** (for Delta Sync queue) from the past 2 hours
2. Look for a pattern of errors from a specific customer or CRM system
3. Determine if there's a backoff active
4. Check if this customer's messages are blocking messages from other customers (given the FIFO and message group limitations)

The challenge is that with SQS visibility timeout, even if you pull a specific message, you might not see it because it's in a backoff state.

---

## CRM System Variability and Integration Challenges

### The Enterprise Customization Problem

Enterprise systems allow customers to configure them in nearly unlimited ways:

> These CRM systems, which are like you can configure absolutely everything and everyone in however way you want it and then they add custom like custom restrictions or custom tables or modify existing queries because essentially every customer CRM system can then more or less be its own integration. I mean like that is not possible to do an integration with if every different environment behaves differently.

**Challenges this creates**:
- Every customer's Enterprise instance can have different configurations
- Custom fields, custom tables, custom validation rules
- Errors that occur for one customer may never occur for another
- Solutions that work for one customer may not work for another
- Requires close collaboration with Enterprise developers to understand customer-specific configuration

Erik expressed frustration with this variability, noting that it makes robust integration design and testing extremely difficult.

---

## Key Takeaways

1. **Integration has strong data recovery capabilities**: Even if real-time sync fails, full sync can restore customers to the correct state. Data loss is extremely hard to trigger, and multiple recovery paths exist.

2. **SQS FIFO message grouping is a critical constraint**: The 20,000-message limit on group scanning can block all processing for all customers if one customer floods the queue. The CRM ID was added to message groups to mitigate this. Monitor queue age (not count) to catch this early.

3. **Database migrations are manual and mandatory**: Before every release, check if delta SQL files changed. If they did, manually apply them. There is no automatic migration system. Missed migrations cause service failures (with fast detection), forcing customer full syncs.

4. **The MA deadlock incident was a watershed moment**: 400,000+ jobs stuck for 2.5 weeks due to a silent HTTP client hang. It highlighted the need for job age monitoring (now implemented) and showed how a shared library bug can cascade across services.

5. **Installation errors are usually CRM-side**: Most "unknown error" responses during setup come from CRM systems. Integration Manager logs are your starting point. You'll often need to contact CRM developers, not debug Apsis code.

6. **Error severity requires judgment**: A 400 response + ambiguous message usually means CRM-side reconfiguration is needed, not Apsis patching. These should be warnings, not critical alerts. Consider what an Apsis engineer can actually fix at 3 AM.

7. **Two queue scenarios need monitoring**: Delta Sync queue (bulk customer data changes) and Outbound Worker queue (high-volume email events) can both spike. Alarms based on message age catch problems faster than count-based alarms.

8. **Log selection is guided by error description**: From the customer's description (installation vs. mapping vs. full sync), you can narrow down which service logs to check, eliminating 90% of noise.

9. **CRM collaboration is essential and varied**: Each CRM (Enterprise, Tribe, E-Deal, Web CRM) has a Teams channel. Enterprise prefers email for support issues. Knowing who to ask for what is critical.

10. **Full sync is the recovery tool of last resort**: It's powerful but expensive on CRM resources. Use it when messages are stuck or data is misaligned, but don't overuse it. It can now be triggered internally.

11. **Dynamics connector had a configuration gap**: The consent timeline feature added events to the generic spec but didn't update the connector loader. Lesson: features that touch bootstrapping must update the configuration file.

12. **OpenAPI-first development improves consistency**: Defining the endpoint spec before implementation avoids CloudFormation/Swagger duplication and ensures documentation is always current.

---

## Unresolved Questions and Action Items

**Erik's action items**:
- Send email to Enterprise team about the campaign creation error observed during the KT session (within the hour)
- Invite Michal, Tomasz, and others to the CRM collaboration Teams channels (FCCRM and Marketing Connect Connector)
- Clean up legacy/unused log groups and Lambda functions to reduce confusion (ping-pong lambdas, old Hello implementations, old full sync queue artifacts)
- Coordinate with product team on priority for the Integration Statistics Service feature
- Consider automating database migrations (bash script or new Golang framework) to remove manual pre-release burden

**Michal/Tomasz action items**:
- Review the two alarm stories created from this session
- Participate in CRM collaboration channels once added
- Consider when to start the Integration Statistics Service project (likely 2-3 weeks out, subject to product/email/SMS/pre-filled form prioritization)

**Uncertain/deferred decisions**:
- Whether to downgrade certain CRM error types from critical alerts to warnings (Erik suggested, but final decision deferred to Michal/Tomasz based on their on-call experience)
- Which framework/approach to use for automating database migrations
- Exact scope and priority of the Integration Statistics Service feature (pending product meeting next week)