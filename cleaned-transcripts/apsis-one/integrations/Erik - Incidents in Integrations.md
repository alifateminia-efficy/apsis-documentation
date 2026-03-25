---
source_file: Erik - Incidents in Integrations.txt
domain: Apsis One Integrations
topics: [SQS FIFO Queue Limitations, Database Migrations, Critical Historical Incident (MA), Installation and Uninstallation Errors, CRM System Integration Issues, Message Queue Management, Error Handling and Logging, Full Sync Recovery, Integration Statistics Feature]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski, Lukasz Grabowski]
key_components: [Integration Manager, Mappings Manager, Full Sync Service, Delta Sync Queue, Outbound Worker Queue, SQS FIFO Queues, AWS Lambda, Message Group Configuration, Exponential Backoff, Generic Spec Configuration]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Outbound Flow, Microsoft Dynamics]
---

## Session Overview

Erik Andersson leads a comprehensive knowledge transfer session on incidents and operational challenges within the Apsis One Integrations domain. The session covers three major incident categories: SQS FIFO queue limitations that can cascade across customers, database migration oversights that require manual intervention, and a critical historical incident in the MA service caused by HTTP client library behavior. The discussion emphasizes recovery mechanisms, logging strategies, and collaboration with CRM system teams. The session concludes with planning for a simple integration statistics feature as a training project.

---

## Overview of Integration System Resilience

### Design Philosophy and Data Loss Prevention

By its nature, the integration system is not highly incident-prone compared to other services. The primary risk consideration is **data loss**, defined not as failed message processing, but as the inability to restore a customer's data to its correct state.

The integration system has multiple built-in safeguards against actual data loss:

- **Full Sync mechanism**: Downloads every contact, all mapped data, and all consent data from the CRM system. This enables recovery from outages, network issues, bugs in Apsis, or bugs in the CRM system.
- **Bidirectional data availability**: Data exists in either Apsis or the CRM system. If the CRM has bugs and doesn't process consents correctly, Apsis can export its superior consent state back to the CRM.
- **Accidental delta sync deployment**: Even if a delta sync is deployed incorrectly and discards real-time syncs, the data still exists in the CRM system and can be recovered via full sync.

> "We have a lot of tools in integration to recover from this."

---

## Critical Issue #1: SQS FIFO Queue Message Group Limitations

### The Core Problem: AWS's 20,000 Message Window

AWS SQS FIFO queues have an implementation limitation that only checks the first 20,000 messages for message group IDs. This creates a cascading failure scenario where one misbehaving customer can block all other customers.

**Historical Context:**

The integration system groups messages in FIFO queues using: `account_section + integration_id + crm_id` (formerly only `account_section + integration_id`).

When a customer performs a large database migration in their CRM system—such as adding a field to every contact—it triggers 200,000+ contact update webhook messages to Apsis. If an error occurs with one of these messages early in the queue:

1. A backoff is placed on that message group
2. 120,000+ messages remain queued in this group
3. AWS only inspects the first 20,000 messages for group IDs
4. The backoff is found only in the first 20,000, placing ALL messages in that group on hold
5. Meanwhile, messages from OTHER customers queue behind these 120,000 messages
6. No processing occurs for any other customer until consumption drops below 20,000 messages

**Resolution and Workaround:**

The affected messages must be blacklisted and discarded in the consumer. A full sync is then required from the customer to ensure data consistency. Processing resumes once the queue is cleared.

### Prevention: Message Group Refinement

The message grouping strategy has been improved to include the **CRM ID** in the group key. This ensures backoffs only affect a specific contact from a specific customer, not the entire customer account.

### Two Primary Risk Queues

#### Delta Sync Queue

This receives real-time messages when customers change their data model. Customers can send massive bursts if they:
- Add new fields to contacts
- Perform bulk updates without disabling webhooks beforehand

**Best Practice**: Customers should temporarily disable webhooks during major database migrations.

#### Outbound Worker Queue

This FIFO queue can receive enormous bursts during email activity syncing. For example, when an email send activity is set up for syncing and a customer has 100,000+ contacts:

- Sent events: 1x recipients
- Delivered events: 1x recipients
- Opened events: 1x recipients
- Clicked events: 1x recipients

A single email campaign with 500,000 recipients can generate 2 million queue messages in rapid succession. Tribe customers have been observed sending approximately **500,000 events** to this queue.

### Queue Monitoring and Alarms

**Current approach**: Message age-based alarms rather than message count.

- Alarm triggers if messages remain in queue for **over 1 hour**
- This avoids false positives during legitimate high-volume operations (like batch exports that add 200,000 messages)
- Check by looking at the oldest message age in the queue rather than total count

When alarms trigger, investigate:
1. How many messages are actually in the queue?
2. Are there errors in the consumer logs?
3. Is there a backoff affecting everything, or is something specifically stuck?

---

## Critical Issue #2: Database Migration Oversights

### Manual Migration Requirement

Database migrations in the integration service are **not automatic**. This is a common source of incidents.

**Migration Files:**

The integration service maintains two SQL migration files:

```
schema_create.sql    # Full database schema (used for fresh setup)
delta_<version>.sql  # Incremental updates (applied before releases)
```

When implementing a new feature that requires schema changes, you apply only the delta file, never the full schema create.

**Critical Process:**

Before any release to beta or production:

1. Review the code changes in the merge request
2. Check if `delta_<version>.sql` was modified
3. If yes, manually apply the delta migration to your local database:
   ```
   <connect to database and run delta SQL file>
   ```
4. Only after migration is applied, approve and merge the PR

**Consequence of Missing Migration:**

Services will fail with database errors, immediately triggering on-call alarms (not silent failures). However, the impact is:
- Customer must run a full sync to restore data consistency
- Operational annoyance and reputation impact
- Retries work after migration is applied (queued messages will eventually succeed)

**Open Challenge:**

Erik has not yet found a clean automatic approach in Golang for managing migrations, though custom bash scripts or newer frameworks might work.

---

## Critical Issue #3: The MA Service HTTP Client Deadlock (Historical)

### The Incident

This represents **the most serious incident the integration team has ever encountered**, affecting not just integrations but the MA (Marketing Automation) service.

**Root Cause:**

Audience (an external system) changed how they handle backoff responses for HTTP 429 (Too Many Requests) errors:
- **Before**: Returned HTTP 429 with a response body
- **After**: Returned HTTP 429 with an **empty response body**

The shared HTTP client library in MA had a bug: when receiving a non-successful (4xx/5xx) response with an **empty request body**, the response was never terminated. The request object remained in a pending state indefinitely.

**Cascade Effect:**

MA has multiple consumer workers that process messages from queues. Each consumer was making HTTP requests to Audience. Over time, every single consumer encountered the 429-with-empty-body response and became stuck indefinitely. With all consumers waiting forever:

- No successful logs were generated
- No error logs were generated
- No page duty alarms triggered (because nothing was happening)

**Discovery:**

Over **2.5 weeks**, 400,000+ jobs accumulated in MA without being processed. The incident was only discovered when investigating queue depths.

### Why This Was Critical for MA

Unlike integrations, MA is **time-sensitive**. Jobs must complete within strict SLAs (e.g., within 1 hour). Example scenario:

A customer sets up a flow: "If this event occurs on a profile, send this discount offer."

If the job processes after 2.5 weeks instead of within 1 hour:
- Customer purchases orders without the promised discount
- Discount arrives too late to be useful
- Customers become angry about broken purchase workflows
- Confirmations arrive weeks late

**Recovery Approach:**

Jobs cannot simply be retried. Each affected customer must be contacted individually for each flow and asked:
- Do you want us to retry this job?
- This requires manual intervention for every affected flow

**Cleanup Effort:**

Erik and Shervidia worked until 9:00 PM for approximately 1.5 weeks to manually process the backlog and restore MA to a stable state.

### Subsequent Improvements: Message Age Monitoring

A **Lambda function** was implemented in MA that continuously monitors queue health:

```
Check: How old is the oldest message currently in the queue?
```

This approach was chosen because:
- You cannot set automatic alarms on message count (false positives during legitimate bulk operations)
- Message age is a better indicator of stalled processing
- Alarms are triggered when messages exceed an expected age threshold

---

## Installation and Uninstallation Error Handling

### Common Patterns

Most customer-facing errors during installation/uninstallation flow through the **Integration Manager** service.

**Explicit error messages** (good):
- Invalid URL: "This URL is bogus. We can't find it."
- Authentication failure: "These credentials are not working. We received a 401."

**Ambiguous error messages** (common):
- "Unknown error occurred. Here is the integration error ID: `<ID>`"

### Typical Failure Points

#### Installation Issues

When registering webhooks with the CRM during installation, ambiguous errors often occur due to:
- Incorrect permission settings in the CRM
- CRM returning internal server errors
- Configuration mismatches

#### Uninstallation Issues

When attempting to delete webhooks from the CRM:
- CRM returns unknown errors during webhook deletion
- Current process: **Uninstallation is blocked** if webhook deletion fails
- Reason: Must ensure no trailing webhooks remain in the CRM to prevent messages being sent to Apsis indefinitely

> "We want to be able to delete all trailing webhooks in the CRM so they don't continue to send things to us."

### Log Navigation Guide

For any installation/uninstallation problem, start with:

**Integration Manager logs** → Only source of truth for setup-related issues

**General principle for all issues:**

The SOC (Support Operations) description usually indicates where the problem occurred. Use this to narrow down which service logs to check:

- Installation/Uninstallation problems → **Integration Manager**
- Mapping save failures → **Mappings Manager**
- Full sync failures → **Full Sync Service** / **Full Sync Producer**
- Real-time sync issues → **Delta Sync Worker**
- Email/activity sync issues → **Outbound Worker**

> "Look at the description of the problem and then depending on that description like you would know is the problem in the inbound flow, is the problem in the outbound flow or is the problem like trying to set up the the the integration so like you can exclude 90% of the logs just from the error description."

Log group naming is descriptive: `integration-manager`, `mappings-manager`, `fullsync-producer`, etc.

---

## CRM System Integration Errors and External Dependency Issues

### Error Categorization: External vs. Internal

Many PagerDuty errors originate from unexpected responses received from CRM systems. These errors are often **not actionable at 3 AM** because they require CRM-side configuration changes.

### Example: Campaign Creation Failures

An alarm may show: `"External system error while creating campaign in Enterprise"`

This indicates:
- Apsis attempted to create or update a campaign/form/activity in the CRM
- The CRM returned an unexpected error response
- This is a **blocking error**: The customer cannot complete their intended action

**Root causes often include:**
- Custom CRM configuration that conflicts with Apsis assumptions
- Character limit violations (e.g., customer named email campaign with length `X`, but Enterprise has character limit `Y`)
- Permission or configuration issues on the customer's CRM instance

**What you cannot do at 3 AM:**
- Deploy a patch to fix CRM behavior
- Reconfigure the customer's CRM system
- Resolve custom instance-specific issues

### Collaboration Channels with CRM Teams

Direct channels exist for each supported CRM system (e.g., Teams channels):
- `#f-crm-and-marketing-connect-e-deal`
- `#f-crm-and-marketing-connect-enterprise`
- `#f-crm-and-marketing-connect-tribe`
- `#f-crm-and-marketing-connect-web-crm`

**Exception**: Enterprise team refuses Teams communication. All questions must be sent via email instead.

### Error vs. Warning Classification

**Current approach (evolving):**

Errors that represent third-party system issues should be downgraded to **warnings** when:

1. They cannot be fixed by a code deployment in Apsis
2. They require CRM reconfiguration or investigation
3. They occur in async/background jobs (not blocking user actions)
4. No on-call engineer can take meaningful action at 3 AM

**Decision logic:**

```
Can I deploy a patch and fix this? → Error
Requires CRM team investigation? → Warning (if appropriate timing)
Is this a blocking synchronous operation? → Error
Can this safely wait until business hours? → Warning
```

**Why this matters:**

If a malformed response from a CRM system triggers an error, and this system handles high-volume asynchronous work (like processing 100s of event syncs daily), on-call engineers would spend nights acknowledging false positives instead of addressing real issues.

---

## Feature Development: Configuration Management for New Connectors

### Incident: Missing Dynamics Configuration

During testing of a new consent timeline feature, a breaking bug was discovered before production release.

**The Problem:**

A new file `generic_spec.go` was created containing:
- Event definitions for the consent timeline feature
- Custom attribute definitions to bootstrap on installation
- These definitions needed to be loaded for every connector that uses them

The configuration is loaded in: `lib/config/connectors.go`

This file registers all available connectors at service startup:

```
These are all of the available integrations that we have.
This is where we specify custom attributes and events to bootstrap on installation.
```

**What was missed:**

Dynamics connector configuration was added to `generic_spec.go` but not registered in the connector configuration file. Result:

- When Dynamics was installed, the new consent events were not bootstrapped
- Consent updates failed because the consent event didn't exist in the system
- Bug discovered during beta testing (before production)

**Fix applied:**

Added Dynamics configuration to the connector initialization. Full consent timeline feature is now deployable to production.

**Lesson:** When adding connector-specific features, ensure both:
1. Feature implementation in connector-specific files
2. Registration in the master configuration file (`lib/config/connectors.go`)

---

## Proposed Feature: Integration Statistics Service

### Problem Statement

Product team repeatedly requests:
- How many customers have installed connector X?
- How many total customers are using connector Y?
- Which sections have connector Z installed?
- Export all installation metadata

Currently, this requires:
- Direct database query access to the integration database
- Manual export and permutation of installation data
- No self-service UI or API

### Proposed Solution: Simple Integration Statistics Endpoint

A minimalist feature to expose installation statistics via API, supporting a new back office UI tab.

**Implementation approach:**

1. Define OpenAPI specification for the new endpoint
2. Generate service interfaces from OpenAPI spec
3. Implement endpoint handler in Integration Manager service
4. Expose in back office with new statistics tab

**Why this is valuable as a training project:**

This simple feature exercises the complete integration development flow:

- **OpenAPI-first approach**: Define contracts before implementation
- **Code generation**: Understand how interfaces are generated from specs
- **Service endpoint implementation**: Add handler to Integration Manager
- **Cross-service communication**: Back office calls integration service
- **Database queries**: Simple data retrieval from installation table

**Estimated effort:** 2-3 days working together (versus half a day solo)

### Why OpenAPI-First Approach?

**Historical context:**

Previously, endpoints were defined in two places:
- CloudFormation files
- Swagger documentation

This caused constant drift and synchronization issues.

**Current approach:**

1. Define API contract in OpenAPI specification
2. **Generate** service interfaces from OpenAPI
3. Implement handlers against generated interfaces
4. CloudFormation automatically reflects the API

**Benefits:**
- Single source of truth for API contracts
- Always up-to-date Swagger documentation
- Reduced CloudFormation clutter
- Type-safe interface generation

---

## Logging and Monitoring Infrastructure

### Log Group Organization

Log groups follow descriptive naming conventions:

- `integration-manager-logs` → Installation, uninstallation, configuration
- `mappings-manager-logs` → Mapping operations
- `fullsync-producer-logs` → Full sync orchestration
- `fullsync-*-logs` → Full sync job-specific logs (legacy)
- `delta-sync-logs` → Real-time sync operations
- `outbound-worker-logs` → Event forwarding to CRM systems

### Legacy Log Cleanup

**Planned cleanup** to reduce confusion:
- Remove unused `ping-pong` lambda logs
- Remove old lambda implementation logs no longer in use
- Delete full sync staging logs from development iterations (approximately 10 pages of SQS queues from testing)

### AutoScaling Alarms False Positives

CloudWatch auto-scaling alarms frequently enter alarm state by default:

> "Just because it says there are like 6 alarms, just make sure to hide the auto scaling alarms because this can be a false positive."

Do not page on-call for auto-scaling alarms without confirming they reflect actual issues.

---

## Exponential Backoff Strategy and Queue Behavior

### Current Backoff Schedule

When messages fail in the outbound worker queue, exponential backoff delays are applied:

```
1st retry:    ~2-3 seconds
2nd retry:    ~1 minute
3rd retry:    ~5 minutes
4th retry:    ~15 minutes
5th retry:    ~1 hour
6th retry:    ~3 hours
7th retry:    ~5-6 hours
Final:        Dead letter queue
```

### Cascading Failure Scenario

When a CRM system is malfunctioning and rejecting all messages:

- **Message 1** in queue: waits 12 hours before 2nd retry
- **Message 2** in queue: waits behind message 1, then gets its own 12-hour backoff
- **Message 998** in queue: could be delayed 12+ hours multiple times

With 1,000 failing messages from one installation:
- Total queue could be blocked for days
- All subsequent customer messages queue behind this
- Cannot process until the bad messages expire to dead letter queue

### Visibility Timeout Complication

SQS has a **visibility timeout** feature: when a message is being processed, it's hidden from other consumers. If processing doesn't complete (backoff), the message reappears after the visibility timeout.

**Operational challenge:** You cannot directly query SQS for messages older than N hours in a specific message group. You must:

1. Check outbound worker logs for the specific message ID
2. Look for error patterns and backoff indicators
3. Determine if the issue is customer-specific or system-wide

---

## Incident Response: Recent Queue Depth Issues

### Example: Old Message in Queue Alert

[Tomasz reported messages aging in the old queue]

**Investigation process:**

1. Check outbound worker logs for the specific message time period
2. Look for error patterns
3. Determine which installation is affected
4. Understand why messages are not progressing

**Likely causes:**
- One installation throwing errors → messages never process
- CRM system returning consistent failures → exponential backoff applies
- Messages eventually reach max retries → moved to dead letter queue

**If caused by CRM malfunction:**
- Contact CRM team (via Teams or email per team preferences)
- Request investigation of customer's CRM configuration
- May require customer to reconfigure their system

---

## Key Takeaways

1. **Integration is resilient by design**: Multiple recovery mechanisms (full sync, bidirectional sync, data existence in both systems) prevent true data loss.

2. **SQS FIFO limitations are real**: The 20,000 message group inspection window can cascade failures across all customers. Message grouping now includes CRM ID to limit blast radius.

3. **Queue monitoring must be age-based**: Alarm on message age, not count. High-volume legitimate operations can create false positives.

4. **Database migrations are manual**: Always check delta files before merging code. Errors are caught immediately but require full sync recovery.

5. **CRM system errors are third-party issues**: Many alarms represent external system problems that cannot be fixed at 3 AM. These should be warnings, not errors.

6. **Start with Integration Manager logs**: 90% of installation/uninstallation problems can be diagnosed from log group names and error descriptions.

7. **Historical lesson from MA**: Empty HTTP response bodies can cause deadlock in shared libraries. Always monitor message age in queues, not just count.

8. **Collaborative CRM debugging**: Direct channels to each CRM team enable quick issue escalation. Enterprise team requires email communication.

9. **OpenAPI-first development**: Define contracts before implementing endpoints. Generates interfaces and keeps documentation in sync.

10. **Simple features teach complex flows**: Building integration statistics service teaches OpenAPI, code generation, endpoint implementation, and cross-service communication.

---

## Unresolved Questions and Action Items

### Immediate Actions

- **Erik to create email tickets to Enterprise team** regarding the two recent alarms (campaign creation errors) and attach Michal and Tomasz to the conversation
- **Erik to prepare production release** for the Dynamics consent timeline feature (already merged to beta)
- **Erik to clean up legacy log groups** (ping-pong lambdas, old implementations, full sync staging logs)

### Future Planning

- **Discuss with Product team** (next week) regarding Integration Statistics feature priority
- **Define Integration Statistics story** and estimate effort once Product prioritizes
- **Schedule 2-3 day pairing session** to implement Integration Statistics together as training project
- **Invite Michal and Tomasz** to CRM team collaboration channels by January for ongoing CRM system communication

### Ongoing

- Monitor for additional old queue messages
- Reassess error vs. warning classification as team handles more incidents
- Explore automatic database migration approaches (bash scripts, new frameworks)