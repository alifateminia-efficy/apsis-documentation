---
source_file: Erik - Incidents in Integrations.txt
domain: Apsis One Integrations
topics: [Incident Prevention and Recovery, Queue Management and FIFO Limitations, Database Migrations, CRM Integration Error Handling, Installation and Uninstallation Workflows, Logging and Troubleshooting, Future Development Planning]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski, Lukasz Grabowski]
key_components: [Integration Manager, Mappings Manager, Full Sync Service, Delta Sync Queue, Outbound Worker, SQS FIFO Queues, Integration Database, Back Office, OpenAPI Specification]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Outbound Flow, Microsoft Dynamics]
---

## Session Overview

This knowledge transfer session covers critical incident patterns, prevention strategies, and operational troubleshooting for the Apsis One Integrations platform. Erik Andersson, the domain lead, discusses three major incident categories: SQS FIFO queue management limitations, database migration oversights, and CRM system communication errors. The session emphasizes that while integrations are relatively incident-resistant by design due to data recovery mechanisms, specific technical gotchas require deep understanding to avoid customer impact. The team also discusses a future statistics service project that would provide visibility into integration installation metrics across the platform.

---

## Understanding Integration Incident Resilience

### Why Integrations Are Relatively Incident-Resistant

Integration is inherently less incident-prone than many other services because data loss is very difficult to achieve. Erik frames this with an important distinction: while data loss might be defined as "failing to process a message," true data loss—the inability to restore a customer's data to the proper state—is much harder to encounter.

**Why recovery is always possible:**

The platform maintains multiple copies of truth:
- Customer data remains in the CRM system (all profiles, contacts, consent data)
- Apsis stores its own copy of synced data
- The **full sync mechanism** can restore any state by downloading every contact and all mapped data fresh from the CRM
- If consent states diverge (e.g., CRM doesn't process consent updates from Apsis), Apsis can export its authoritative consent state back to the CRM

[Erik Andersson]: > "By nature it is quite hard to encounter real data loss in integration... we have a lot of tools in integration to recover from this."

This recovery capacity exists even when:
- Bugs are introduced in Apsis code
- The CRM system has bugs that prevent processing
- Network outages occur
- Incorrect delta sync configurations are deployed

---

## Critical Incident Pattern #1: SQS FIFO Queue Bottlenecks

### The 20,000 Message Group Limitation

AWS SQS FIFO queues have a critical implementation detail that can cause platform-wide cascading failures: **AWS only checks the first 20,000 messages in a queue when looking for message group IDs.**

#### Historical Context: The Problem Case

The integration team encountered a severe incident where:

1. **Initial message grouping strategy:** Messages were grouped only by `account_section` + `integration_id`
2. **Triggering event:** A customer made a major real-time sync change—they added a new field to every contact in their CRM and triggered ~200,000 contact update messages
3. **The cascade:** 
   - One of these messages failed (for any reason)
   - AWS placed a backoff on that message group
   - 120,000 messages accumulated behind the failed message
   - AWS only scanned the first 20,000 messages for group IDs
   - The backoff was found and applied only to the first 20,000
   - The remaining 100,000+ messages were stuck behind, invisible to the consumer
   - **Every other customer's messages were now blocked** because they couldn't get past the 120,000 backlog

#### The Response: Blacklisting and Full Sync

When this occurred, the team had to:
1. Implement a blacklist in the consumer that discarded all messages from the problematic customer
2. Notify SOC that the customer needed to run a full sync to restore data consistency
3. Wait for the queue to drain below 20,000 messages before normal processing resumed

#### Mitigation: Improved Message Grouping

The team changed the message grouping strategy to include the CRM ID as well:
- **New approach:** `account_section` + `integration_id` + `crm_id`
- **Result:** Backoffs are now scoped to individual contacts, not entire customer accounts
- **Impact:** A customer's data model migration no longer blocks all other customers

### Two Critical Queues to Monitor

#### 1. Delta Sync Queue (Inbound Flow)

This queue receives real-time webhook messages whenever data changes in the CRM system. Customers can accidentally trigger massive bursts by:
- Adding a new field to all contacts without disabling webhooks first
- Running bulk data migrations
- Schema changes that cascade to every record

**Mitigation:** Always ask customers to temporarily disable webhooks before major CRM migrations, though this is often forgotten.

#### 2. Outbound Worker Queue (Outbound Flow)

This FIFO queue handles events being synced from Apsis back to the CRM. Email sendings are the primary concern:

> "If you haven't set up an email activity to be synced and the customer has like a couple of tens of thousand or even 100,000 of contacts, then we can receive like 2 million messages in this queue very, very swiftly."

**Why email events cause floods:**
- Each email send generates: `sent`, `delivered`, `opened`, `clicked`, `viewed` events (approximately 5 events per recipient)
- A 500,000-contact email campaign from a large customer can generate 2.5M+ messages instantly
- The Tribe platform, for example, can push ~500,000 events into this queue

**Current mitigation:**
- The team groups many events together before processing, which helps significantly
- They implemented **message age-based alarms** (see below)

### Queue Monitoring: The Age Alarm Strategy

Because you cannot predict message volume spikes (a 200,000-message dump during an export is normal), the team doesn't alarm on message count. Instead:

**They implemented a Lambda that continuously checks:** How old is the oldest message in the queue?

- **Trigger threshold:** Messages older than ~1 hour generate an alarm
- **Why this works:** It indicates messages are stuck, not just backlogged
- **Investigation approach:** When the alarm fires, check:
  - Consumer logs for errors
  - Whether a backoff is stuck on a particular message group
  - If something has completely stopped consuming

---

## Critical Incident Pattern #2: Missed Database Migrations

### The Migration System Architecture

Integration uses a **manual, two-file migration approach:**

1. **`schema_create.sql`** — The complete database schema, used only for fresh installations
2. **`delta.sql`** — Incremental migration files that add features and updates

When implementing new features that require database changes:
- You add the schema changes only to `delta.sql`, never to `schema_create.sql`
- These must be applied manually before releasing to beta or production

### The Problem: Migrations Are Not Automatic

[Erik Andersson]: > "The migration in integration is not automatic... it's important to run this migration before you do any release like check in the files changed before you click on merge."

**Current process:**
1. You make code changes and schema changes
2. Before creating a release, you must manually:
   - Set up a database connection to the target environment
   - Run the corresponding SQL files from `delta.sql`
3. Only then can the service startup successfully

**Why it fails:**
Developers often forget this manual step, which leads to immediate failures when the service starts and tries to access tables that don't exist yet.

### Failure Impact: Annoying But Recoverable

When a migration is skipped:
- **Services fail immediately** — This gets caught fast by on-call/paid duty
- **Symptoms:** Services crash at startup with column/table not found errors
- **Recovery:** 
  - Apply the migration
  - Services restart successfully
  - Queued messages retry automatically
  - No data is lost

**The real cost:** Having to ask customers to run full syncs to restore consistency if they happened to trigger operations during the outage.

### Recommendations for Improvement

[Erik Andersson]: > "I very much encourage you if to see if you can find any cool way of doing this automatically... we've failed so far to find any nice automatic way to do this in Golang, but it might be possible like doing it with some custom bash script or if there has been any new framework released to handle this."

This remains an open problem the team would like to solve but hasn't found a reliable automated solution in their Go/AWS stack.

---

## Critical Incident Pattern #3: Silent Service Degradation — The MA 429 Response Body Bug

### The Most Serious Incident in Integration Team History

[Erik Andersson]: > "By far the most serious incident we've ever had to handle in our team was actually for integration... it was for MA, but I think you might have also actually been impacted by this in one."

This incident provides crucial context for why monitoring and alerting strategy matters.

#### What Happened

1. **The Change:** Audience (an external CRM system) modified how they handle rate-limiting responses
   - **Before:** Returned HTTP 429 with a response body
   - **After:** Returned HTTP 429 with an **empty response body**

2. **The Code Problem:** MA had a shared HTTP client library that exhibited a critical bug:
   - When it received a non-successful HTTP response with an **empty request body**, the function **never returned**
   - No response object was created
   - No error was logged

3. **The Cascade:**
   - MA services contain worker pools that consume messages from queues
   - When each worker encountered a 429 with empty body, its request hung indefinitely
   - Over time, **every single worker in the service got stuck** on a hanging request
   - New messages arrived in the queue but had no available workers to process them

4. **The Blindness:** This wasn't detected for **2.5 weeks** because:
   - No errors were logged (the function never returned, so exception handlers never fired)
   - No successful responses were logged
   - Standard error alarms were silent
   - The system appeared to be working fine; it just wasn't processing anything

5. **The Discovery and Impact:**
   - After 2.5 weeks, discovery revealed **400,000+ jobs stuck in MA**
   - MA processes are time-sensitive: discount offers, purchase confirmations, and event-triggered workflows must execute within ~1 hour to be valuable
   - A job sitting for 2.5 weeks is worthless—the customer likely never received their promised discount or confirmation
   - You cannot simply retry those jobs; they need customer approval

6. **The Recovery Effort:**
   - Required manual individual outreach to each customer for each affected flow
   - Had to ask: "Should we retry this operation from 2.5 weeks ago?"
   - Erik and Shervidia worked until 9 PM every day for **1.5 weeks** just to clean up

### The Lesson for Integrations

While Integrations doesn't have the same strict time-sensitivity as MA, the lesson is critical: **Don't rely solely on error logging to detect failures.**

#### Changes Made Since Then

The team implemented a **custom Lambda** that continuously monitors job staleness:

```
Lambda checks:
  - How old is the oldest message in the job queue?
  - Not: How many messages are queued?
  - Reason: You can't predict legitimate spikes (200k message exports are normal)
  - But: A message older than expected indicates the queue is stuck
```

This Lambda generates alerts based on **message age**, not message count, which allows the team to detect stuck jobs within hours rather than weeks.

---

## Error Handling and Support Workflows

### Installation and Uninstallation Errors

By far the most common customer-visible issues occur during setup and teardown. The team categorizes errors into two types:

#### Explicit, Customer-Actionable Errors

These have clear error messages:

- **Invalid CRM URL:** "This URL is bogus; we can't find it. You entered an incorrect URL."
- **Authentication failure:** "These credentials are not working. We received a 401 from the CRM."

These errors allow customers to immediately correct the problem.

#### Ambiguous Errors Requiring Investigation

For most other errors, the system returns:
> "Unknown error occurred. Here is the integration error ID: [ID]"

[Erik Andersson]: > "This can happen during installation time when we try to register webhooks because it might be that the permissions were not properly set or something and the CRM returns an internal server error like of course we have no idea what that actually means. The CRM people need to investigate that."

**Uninstall errors are particularly problematic:**
- If uninstallation fails with an ambiguous error, the system **blocks the uninstallation** to prevent orphaned webhooks
- Orphaned webhooks would continue sending data to Apsis even after the customer has disconnected
- This puts the customer in a stuck state where they can't uninstall

### Logs for Troubleshooting: The Service Mapping

When a customer reports an issue, the log location is usually obvious from the error description:

| Customer Report | Check These Logs | Notes |
|---|---|---|
| "Failed to install integration" | Integration Manager logs | Installation setup and webhook registration |
| "Failed to save mappings" | Mappings Manager logs | Field and activity mapping configuration |
| "Full sync failed" | Full Sync service job logs | Data synchronization issues |
| "Failed to uninstall" | Integration Manager logs | Webhook deletion problems |
| "Real-time sync not working" | Delta Sync logs | Ongoing data updates |
| "Campaign/Form not syncing" | Outbound Worker logs | Activities being sent to CRM |

[Erik Andersson]: > "Based on the description of the problem you would know: is the problem in the inbound flow, is the problem in the outbound flow, or is the problem trying to set up the integration? You can exclude 90% of the logs just from the error description."

**SOC (Support Operations Center) usually provides:**
- Detailed problem descriptions
- Often a direct link to the relevant log
- When they don't, you can infer the service from the symptom

### Handling CRM-Side Errors

A huge portion of pager duty incidents are actually issues in the CRM system that Apsis cannot fix.

#### The Pattern: External System Errors

When Apsis receives an unexpected error response from a CRM system:
- **HTTP status:** 400, 403, 500, etc.
- **Error message:** Vague ("Unknown field," "Permission denied," "Configuration error")

**Current behavior:**
- Apsis logs this as an error
- Page duty alarm fires
- On-call engineer investigates
- **Conclusion:** Nothing can be fixed on the Apsis side; the CRM system needs to reconfigure

#### Example Cases

[Tomasz Kowalski describes an actual incident]: An email campaign creation failed with an error from Efficy Enterprise. The customer named the email with an excessively long name, and Enterprise has an undocumented character limit on campaigns. Enterprise refuses the request, Apsis can't work around it, the customer must shorten the name.

**The Problem with Current Alerting:**
- These are legitimately errors (the operation failed)
- But they are **not operational errors** that an engineer can fix at 3 AM
- They require CRM vendor investigation or customer reconfiguration
- Page duty alerts should not fire in the middle of the night for issues requiring vendor contact

### Evolving the Error/Warning Classification

[Erik Andersson]: > "Is this something I personally can't take action on in the middle of the night? Is this something where I can deploy a patch and it is fixed? And in this case, no, I can't because even if I... there is nothing I can patch to make this work in the CRM system."

**The decision framework for demoting errors to warnings:**

1. **HTTP Status alone is not sufficient** — A 400 can mean "bad request from our code" (fixable) or "your CRM rejected this" (not fixable)
2. **The error message matters** — Only the combination of status + message indicates root cause
3. **Can you take action at 3 AM?**
   - If yes → Error (page duty)
   - If no, but someone should know → Warning (logged, monitored, not paged)

**Current strategy:** Erik is reclassifying many CRM-originated errors from errors to warnings to reduce noise for the on-call engineer. This is an ongoing effort.

### CRM Vendor Collaboration Channels

The team maintains direct collaboration channels with each CRM vendor:

**Established channels:**
- **Efficy E-deal team** — Teams channel
- **Efficy Enterprise team** — **Email only** (they refuse Teams; all support issues must go via email)
- **Tribe team** — Teams channel
- **Web CRM team** — Teams channel

[Erik Andersson]: > "For any enterprise questions, we have to send them an email... if it is a question like how does this work in enterprise or we are planning to do the change, how would enterprise be affected? Then we can put that here in the chat. But if it is anything we received this error for this specific customer, so it's more like support related then they want that in an email instead."

**Process when a CRM error occurs:**
1. Log the issue with full context (customer, request, error response)
2. Send vendor-specific inquiry:
   - For Enterprise: Email with error details, ask them to investigate
   - For others: Teams message in the collaboration channel
3. Wait for vendor response before escalating to customer
4. Often turns out to be a customer-specific configuration issue

---

## Operational Troubleshooting: Log Navigation and CloudWatch

### Log Group Organization

Log groups follow a descriptive naming convention that makes it easy to find the right logs:

- `integration-manager-logs` — Installation/uninstallation, configuration management
- `mappings-manager-logs` — Field mapping and activity mapping operations
- `fullsync-producer-logs` — Full synchronization jobs
- `delta-sync-logs` — Real-time webhook processing
- `outbound-worker-logs` — Sending activities back to CRM

[Erik Andersson]: > "The log groups are rather descriptive of what they are doing."

### CloudWatch Cleanup Tasks

The team maintains several legacy log groups and CloudWatch artifacts that create confusion. Erik notes these will be removed to reduce noise:

- **Ping-pong Lambdas** — Old test functions no longer used
- **Legacy Lambda implementations** — Earlier versions of services still cluttering the console
- **Full Sync SQS queues** — When the new full sync architecture was developed, a new queue was created for each full sync job, and these queues were never cleaned up. In staging, there are **10+ pages of unused queues** from these historical full sync executions

### False Positive Alarms: Auto-Scaling

CloudWatch auto-scaling alarms sometimes fire in alarm states by default for unclear reasons.

[Erik Andersson]: > "Just because it says there are like 6 alarms, we do have auto-scaling and I don't know for whatever reason like that auto-scaling gets in an alarm state by default. I don't really know why that is, but just because it says there are like 6 alarms, we do have auto-scaling. Just make sure to hide the auto-scaling alarms because this can be a false positive."

**Action:** Filter out auto-scaling alarms when reviewing the dashboard; they are not indicative of actual problems.

---

## Recent Bug Fix: Dynamic Connector Configuration

### The Issue: Missing Connector Bootstrap Configuration

While testing the new consent timeline feature, Erik discovered a critical bug in the Dynamics connector configuration.

#### The Problem

**File:** `lib/config/connectors.go`

This file serves as the system's inventory of available connectors and specifies what needs to be bootstrapped during installation:

```go
// On startup, this file defines:
// 1. All available integrations
// 2. Custom attributes to bootstrap for each connector
// 3. Custom events to bootstrap for each connector
```

For the new consent timeline feature:
- Generic event definitions were added to `generic_spec.go` 
- These events need to be registered on the customer's CRM during installation
- But the Dynamics connector was **not added to the bootstrap configuration** in `lib/config/connectors.go`

#### The Impact

- When Dynamics was installed, the consent timeline events were never bootstrapped
- During consent update operations, the system tried to add the consent event to the customer's profile
- But the event didn't exist in the CRM, so every consent update failed
- This would have been a major outage if released to production

#### Discovery and Fix

Erik found this during testing on beta. The fix was simple: add Dynamics to the connector configuration so its bootstrap includes the generic spec events.

Status: Merged to beta, preparing production release. No further testing needed.

---

## Future Development: Integration Statistics Service

### The Need

Product and leadership continuously request data that only exists in the integration database:

- How many customers have installed a specific integration?
- Which accounts/sections have a particular integration installed?
- What's the total integration user base across all CRMs?
- Custom queries and permutations of installation data

**Current workaround:** Someone manually exports from the integration database and manipulates the data in Excel/Sheets.

### Proposed Solution: Simple Integration Statistics Service

Erik proposes building a minimal statistics service that would:

1. **Provide an HTTP endpoint** from the Integration service that returns:
   - List of all installations (account + section + integration combinations)
   - Installation counts by integration, account, section
   - Filtered queries by various criteria

2. **Store data from:** The integration database's installation records (which already track this)

3. **Expose via:** Back Office UI with a new tab where product can query and export this data

[Erik Andersson]: > "Back office would call the integration endpoint to say please give me all integrations. So we still need to add that endpoint... it will touch upon every required component because it will force you to see like how do I add a new endpoint in integration and that will touch upon our OpenAPI specification."

### Why This Is a Good Teaching Project

Despite being simple to implement (Erik estimates 4-6 hours for him alone), this project touches every critical part of the integration architecture:

1. **OpenAPI specification** — Define the endpoint contract first
2. **Generate server interfaces** — Auto-generated from OpenAPI spec
3. **Implement the handler** — Add the actual function to Integration Manager
4. **Database query** — Query the installation table
5. **Return structured data** — Serialize and return properly

**Teaching value:** By walking through this together, team members learn the entire workflow from spec-first design through implementation.

### Timeline and Prioritization

- **Decision needed:** Product and Daniel (architecture) must agree this is worth doing
- **Expected start:** ~2 weeks from this session (after current priorities)
- **Estimated effort:** 2-3 days for thorough walkthrough and review
- **Status:** Not yet on the backlog; needs product sign-off

---

## SQS and Queue Management Deep Dive

### Exponential Backoff Strategy and Its Limitations

When a message fails to process (e.g., CRM returns an error), the integration uses exponential backoff:

**Backoff schedule:**
- Attempt 1: Retry immediately
- Attempt 2: 2 seconds
- Attempt 3: 1 minute
- Attempt 4: 5 minutes
- Attempt 5: 15 minutes
- Attempt 6: 1 hour
- Attempt 7: 3 hours
- Attempt 8+: 5-6 hours

Then eventually: Dead Letter Queue or message discard

#### The Problem with This Strategy

Imagine a CRM system is misconfigured and rejects all messages:

1. **Message 1** fails → Gets 5-6 hour backoff
2. **Message 2** enters queue → Also fails → Gets 5-6 hour backoff
3. **Message 3-1000** → Same pattern

Result: A customer with 1000 failed messages has to wait 5-6 hours before even **trying** message 998, and if the CRM is still misconfigured, message 998 gets another 5-6 hour backoff.

The queue becomes essentially blocked for that customer, and all subsequent messages from other customers wait behind the blocked backoff.

[Erik Andersson]: > "This has been discussed a lot of time if we should change that strategy or not... if the problem still is there, I mean like then you again will have like a 12 hour delay for that message and then you can picture like message number 998 there in the queue, if all of them before will have like a 12 hour backup on it."

**Why this is especially problematic with Enterprise:**

> "I ******* hate Enterprise. But like this, this is the problem with these CRM systems, which are like you can configure absolutely everything and everyone in however way you want it and then they add custom like custom restrictions or custom tables or modify existing queries... every customer CRM system can then more or less be its own integration. I mean like that is not possible to do an integration with if every different environment behaves differently."

### SQS Limitations: Why You Can't Query Message Groups

AWS SQS FIFO queues don't provide an API to:
- Query all messages with a specific message group ID
- Get a count of messages for a specific group
- See which groups have messages stuck

[Erik Andersson]: > "One annoying thing with SQS is that you cannot do a request and say these are the amount of messages in the queue with this message or with this message group. You can just see like there is a message in the queue that is that is this old."

**Workaround:** Check the outbound worker logs directly to see if a specific message group has an error and backoff.

---

## Real Incident Response: The Enterprise Configuration Problem

### The Incident

Two alarm stories were created from actual incidents. One involved an error syncing email campaigns to Efficy Enterprise:

```
Error: External system error
HTTP Status: 400
Error details: [Obscure configuration message]
```

### Investigation Process

1. **Recognize the pattern:** External system error (400) likely means CRM-side issue
2. **Check the logs:** Integration service logs show the request was well-formed
3. **Escalate to vendor:** Email Enterprise team with full error context
4. **Wait for investigation:** They must look at their system configuration
5. **Likely outcome:** Customer has an undocumented configuration limit, field name length restriction, or permission issue

### Message Age in Queue Inspection

When investigating old messages in queues:

The team checked the `outbound_worker` logs from the past 2 hours:

```
Recent messages show exponential backoffs
Pattern: 2 seconds → 1 minute → 5 minutes → 15 minutes → 1 hour → 3 hours → etc.
```

This indicates messages are failing at the CRM and backing off, which is normal recovery behavior—not a stuck queue.

**The key distinction:**
- **Fresh (recent) backoff messages** = Normal retry behavior, probably transient CRM issue
- **Very old messages in backoff** = Something is stuck and needs investigation

---

## Key Takeaways

1. **Recovery is built-in:** The full sync mechanism means true data loss is nearly impossible. Always feel confident recommending a full sync when needed.

2. **FIFO queue management is critical:** The 20K message group limitation and message grouping strategy are essential to understand. Include CRM ID in your message group IDs to prevent customer-wide blocks.

3. **Monitor message age, not volume:** Don't alarm on queue depth; alarm on how long messages have been sitting idle. A 500K message surge is normal; a 2-hour-old message is concerning.

4. **Database migrations require discipline:** Delta migrations are manual; always check if `delta.sql` changed before releasing. Missing migrations fail fast, which is good, but annoying.

5. **Most incidents are CRM-origin issues:** The majority of pager duty alerts are from external CRM systems behaving unexpectedly. Reclassify these from errors to warnings to reduce false urgency.

6. **Error messages matter more than HTTP status:** A 400 from your code is different from a 400 from the CRM. Only the combination of status + message tells you if you can fix it.

7. **Log navigation is straightforward:** Service names match log group names. Read the customer's error description to narrow down which log group to check.

8. **CRM vendors have different communication preferences:** Enterprise requires email; others use Teams. Know which channel to use for each vendor.

9. **CloudWatch has legacy cruft:** Ignore auto-scaling false alarms; plan to clean up old Lambda test functions and full sync queues.

10. **Spec-first API design is valuable:** Define OpenAPI specs first; generate interfaces from them; implement handlers last. This keeps specs in sync and reduces duplication.

---

## Unresolved Questions and Action Items

### Immediate Action Items

- **Erik:** Create email to Efficy Enterprise team with the external system error details from the two incidents; CC Michal and Tomasz
- **Erik:** Clean up legacy CloudWatch artifacts:
  - Remove ping-pong test Lambdas
  - Remove old Lambda implementations
  - Delete all full sync test queues from staging (10+ pages)
- **Team:** Decide on error-to-warning reclassification criteria for CRM-origin errors and implement systematic review

### Near-Term (2+ weeks)

- **Product + Architecture team:** Prioritize the integration statistics service (simple project, high teaching value)
- **Team:** Once product approves, begin implementation as a structured learning project for Michal and Tomasz

### Longer-Term Improvement

- **Database migration automation:** Investigate automatic migration application in Go or via custom bash scripts to eliminate manual step before releases
- **Exponential backoff strategy review:** Consider whether the current aggressive backoff schedule is optimal for problematic CRM systems
- **Error classification framework:** Formalize the decision tree for when external system errors should be warnings vs. errors

### Open Technical Debt

- Full cleanup of CloudWatch log groups and SQS queues from test and legacy implementations
- Auto-scaling alarm false positive root cause investigation

---

## Technical References

### Key Files and Locations

- **Connector Configuration:** `lib/config/connectors.go` — Defines all available integrations and their bootstrap requirements
- **Generic Event Specs:** `generic_spec.go` — Event and attribute definitions used by multiple connectors
- **Database Migrations:** `delta.sql` — Incremental schema changes; must be run before releases
- **Postman Collection:** Available for testing integration endpoints (alternative to curl)

### Services and Log Groups

| Service | Log Group | Purpose |
|---------|-----------|---------|
| Integration Manager | `integration-manager-logs` | Installation, uninstallation, configuration |
| Mappings Manager | `mappings-manager-logs` | Field and activity mapping |
| Full Sync | `fullsync-producer-logs` | Complete data synchronization |
| Delta Sync | `delta-sync-logs` | Real-time webhook processing |
| Outbound Worker | `outbound-worker-logs` | Sending activities to CRM |

### SQS Configuration

- **Delta Sync Queue (Inbound):** FIFO with message group = `account_section` + `integration_id` + `crm_id`
- **Outbound Worker Queue:** FIFO for sending activities back to CRM
- **Message Age Alarm Lambda:** Checks oldest message age (not message count)