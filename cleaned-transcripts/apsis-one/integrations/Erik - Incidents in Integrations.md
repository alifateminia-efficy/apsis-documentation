---
source_file: Erik - Incidents in Integrations.txt
domain: Apsis One Integrations
topics: [SQS FIFO Queue Limitations, Database Migrations, Critical Incident Case Study (MA Service), Installation/Uninstallation Errors, CRM System Integration Challenges, Error Classification and Alerting Strategy, Integration Statistics Service Planning]
speakers: [Erik Andersson (Integration Lead), Michal Rosikiewicz, Tomasz Kowalski, Lukasz Grabowski (Transcriber)]
key_components: [SQS FIFO Queues, Delta Sync Manager, Full Sync Service, Integration Manager, Mappings Manager, Full Sync Producer, Outbound Worker, MA (Marketing Automation), Integration Database, CloudFormation, OpenAPI Specification, Generic Spec Configuration]
session_type: knowledge-transfer
subdomains: [Architecture, Duplicate profiles in Apsis]
---

## Session Overview

Erik Andersson led a comprehensive knowledge transfer session on incident management and architecture patterns in the Apsis One Integrations domain. The session covered the three most critical incident patterns the team has experienced: SQS FIFO queue limitations that can block message processing across multiple customers, database migration oversights that cause cascading failures, and a catastrophic case study from the MA (Marketing Automation) service where stuck HTTP requests left over 400,000 jobs unprocessed for 2.5 weeks. The discussion also covered how to diagnose installation/uninstallation errors, how to collaborate with CRM development teams on integration issues, and planning for a new integration statistics service.

---

## Understanding Integration Incidents and Data Loss Scenarios

### Why Integration Is Incident-Resistant by Design

Integration, by nature, is not highly incident-prone compared to other services. The defining characteristic that makes it resilient is the **existence of recovery mechanisms** rather than the absence of problems.

[Erik Andersson]: > "What is the most dangerous case of an incident? Well, data loss is the worst case. But by nature it is quite hard to encounter real data loss in integration depending on how you define it."

The key distinction is between two definitions of data loss:
1. **Narrow definition**: Failed to process a message
2. **Broad definition**: Unable to restore the customer's data to the proper state

Under the broad definition, data loss recovery is always possible because **data exists in multiple places within the Apsis ecosystem**.

### Multi-Location Data Architecture

Data resilience comes from the architecture's inherent redundancy:

- **CRM System**: Contains the source of truth for all contact profiles, consent data, and custom fields
- **Apsis Platform**: Contains synchronized versions of all contacts and consent states
- **Full Sync Mechanism**: Can download every contact, all mapped data, and all consent states from either system

This architecture enables recovery from:
- Outages in either system
- Network failures
- Bugs in Apsis code
- Bugs in the CRM system
- Accidental incorrect deltas deployed to Sync Manager (which would discard real-time syncs)

### Bidirectional Sync Advantages

Because integration syncs data **both directions** (Apsis → CRM and CRM → Apsis), consent state mismatches can also be resolved:

> "If the CRM doesn't process consents from us and Apsis has a better consent state than the CRM, we can export the consent state from Apsis to the CRM. Data exists in either Apsis or the CRM system."

---

## Critical Incident #1: SQS FIFO Queue Message Group Limitations

### The AWS SQS FIFO Constraint

This is the most common recurring incident pattern. **AWS SQS FIFO queues only check the first 20,000 messages when determining message groups for deduplication and ordering.**

[Erik Andersson]: > "If AWS only looks at the first 20,000 messages for message group IDs, and you have 120,000 messages queued with a backoff on one group, then every new message from any other customer gets stuck behind those messages until you consume down below 20,000."

### How This Incident Manifests

**Scenario**: A customer makes a major real-time sync change to their CRM metadata—for example, adding a new field to every contact. If they have 200,000 contacts, this triggers 200,000 contact update messages flooding the queue.

**Failure cascade**:
1. A bug or error occurs processing one of these messages
2. The message group gets a backoff applied
3. AWS only scans the first 20,000 messages and finds this backoff group
4. Since the customer's messages extend beyond position 20,000, AWS cannot determine if new messages belong to this group or not
5. **All messages for all other customers get held indefinitely** because they're positioned after the problematic customer's messages
6. No new processing happens until the queue drains below 20,000 messages

### The Resolution: Blacklisting During Cleanup

When this occurred, the team had to:

1. **Blacklist the problematic installation** in the consumer code with an explicit check:
   ```
   if message.comes_from(installation_id) {
       discard_message()
   }
   ```
2. **Discard all queued messages** for that customer
3. **Notify SOC** that the customer flooded the system and must perform a full sync
4. **Resume processing** once the queue dropped below 20,000 messages

> "We had to put a blacklist on this specific installation, discard all messages for that customer, and inform SOC they need to do a full sync to ensure all contacts and consent are in the correct state."

### Architectural Improvements: Finer-Grained Message Grouping

The team has since changed the message group key strategy to include the **CRM ID** in addition to account section and integration ID:

**Before**:
```
message_group = account_section + integration_id
```

**After**:
```
message_group = account_section + integration_id + crm_id
```

This change means a timeout on a specific contact's messages no longer blocks messages for other contacts from the same customer.

### Two Vulnerable Queues

#### 1. Delta Sync Queue (Inbound Flow)
- Receives real-time webhook messages when CRM data changes
- **Risk**: Customers can trigger massive bursts by changing their data model and adding fields to all contacts without disabling webhooks first
- **Current safeguard**: Finer-grained grouping by CRM ID reduces impact

#### 2. Outbound Worker Queue (Outbound Flow)
- Processes events going from Apsis back to the CRM
- **Risk**: Email sending activity events create exponential message volume
  - If a customer sends to 100,000 contacts with tracking enabled:
    - Sent event: 100,000 messages
    - Delivered event: 100,000 messages
    - Opened event: 100,000 messages
    - Clicked event: 100,000 messages
    - **Total**: ~300,000-400,000 messages
  - Tribe customers, for example, can generate ~500,000 events in the queue very rapidly

### Monitoring: Message Age Alarm

> "You can't set up automatic alarms on the amount of messages, nor would you want to—you might trigger an export and suddenly have 200,000 messages in the queue."

Instead, the team monitors **message age**:

```
if oldest_message_age > 1_hour {
    trigger_alarm()
}
```

When this alarm fires, the on-call engineer must:
1. Check CloudWatch for the queue
2. Review consumer logs for errors or backoffs
3. Determine if messages are stuck (backoff) or if something has gotten stuck in code
4. Investigate whether the issue is across all customers or isolated

---

## Critical Incident #2: Missed Database Migrations

### Migration Architecture

Integration uses a **two-file database migration system**:

1. **`schema_create.sql`**: The complete database schema, used only for greenfield setups
2. **`delta.sql`**: Incremental migration files for feature releases

When implementing a new feature that requires schema changes, you apply only the delta file, not the full schema.

### The Problem: Automatic Migrations Don't Exist

> "The migration in integration is not automatic. You have to log in, set up a database connection, and run the corresponding SQL files manually."

**Where migrations are required**: Before every release to beta or production.

**Gotcha**: It's easy to forget this step because the system doesn't enforce it automatically (unlike some frameworks).

### What Happens When Migrations Are Skipped

```
Service attempts to use column or table that doesn't exist
  ↓
Service crashes with database error
  ↓
Paid duty alarm triggers
  ↓
Customer must run full sync (expensive operation on their CRM)
```

**Impact**: Medium—services fail fast with errors caught in paid duty, but it's annoying and requires customer action.

### Mitigation Strategy

Before merging any PR to production or master:

1. **Check the PR diff** for changes to the `delta.sql` file
2. **If delta files changed**: Verify you've applied the migration to your local/staging database
3. **Before merge**: Apply the delta SQL to your database connection

[Erik Andersson]: > "Really make sure to apply this before you do any release. It's important to run this migration before you do any release—check the files changed before you click merge."

### Why Full Automatic Migration Hasn't Been Implemented

The team has tried to find an automatic solution but hasn't found a good approach in **Golang**. Possible paths forward include custom bash scripts or newer migration frameworks, but none have been adopted yet.

---

## Critical Incident #3: The MA Service HTTP Client Bug (Catastrophic Case Study)

This was the most serious incident the integration team has ever handled.

### The Trigger: External API Change

**Efficy audience** changed how they handle rate-limit (429) responses:
- **Before**: Returned HTTP 429 with response body
- **After**: Returned HTTP 429 with **empty response body**

### The Bug: HTTP Client Hang

The MA service uses a **shared HTTP client library** with a critical flaw:

> "If the HTTP client received a non-successful response with an empty request body, then the request was never terminated—no response was returned."

When a 429 response came back with no body:
- The HTTP response never completed
- The function never returned
- No success logs were generated
- No error logs were generated
- **The request hung indefinitely**

### Cascade: Complete Queue Stagnation

```
Customer event triggers request to Efficy
  ↓
Efficy returns 429 with empty body
  ↓
HTTP client hangs (never returns)
  ↓
Consumer worker thread blocked (waiting for response)
  ↓
New messages arrive and wait for worker
  ↓
All consumer workers eventually hang
  ↓
Queue processing stops completely
```

### Detection Failure

This was **not detected for 2.5 weeks** because:

- No successful logs (request never returned)
- No error logs (client was hung, not erroring)
- **Page duty was not triggered** (alerting only fires on explicit errors)
- The queue appeared to be processing normally (messages were in it)

### The Scale of Impact

```
Over 400,000 jobs stuck in MA queue
All jobs unprocessed for 2.5 weeks
```

### Why MA Made This Catastrophic

Unlike integration, **MA (Marketing Automation) is time-sensitive**:

- Flows are event-driven: "If customer does X, send them offer Y"
- Events have implicit expiration windows (typically 1 hour)
- If a discount offer is sent 2.5 weeks late:
  - Customer never gets it when they needed it
  - Customer may have already made the purchase
  - Customer experience is severely degraded
  - Messages become meaningless ("Here's your discount from 2.5 weeks ago")

### Recovery: Manual Case-by-Case Review

> "You can't just rerun those jobs. You have to contact each and every customer for each and every flow and ask: Do you want us to retry this or not?"

The cleanup took **1.5 weeks with two engineers** (Erik and Shervidia) to:
1. Contact each customer with affected flows
2. Get approval for retry or discard
3. Manually process decisions
4. Verify queue cleared

### Post-Incident: Message Age Monitoring

A **Lambda function** was implemented to continuously monitor job queue health:

```
Lambda runs periodically
  ↓
Queries: How old is the oldest job in the MA queue?
  ↓
If oldest_job_age > expected_threshold:
    trigger_alarm()
```

This cannot be done based on **message count** because:
- Legitimate bulk exports can create 200,000+ messages at once
- You don't want a page duty alarm every time a big job runs
- **Age is the real metric**—if messages are flowing, old ones should be processed

---

## Installation and Uninstallation Error Handling

### Where Installation/Uninstallation Logic Lives

> "Anything connected to installation and uninstallation is handled in the integration manager. Always start by checking the integration manager logs."

The **integration manager** is the single source of truth for installation/uninstallation flows.

### Error Categories with Clear Messages

Some errors are explicit and actionable:

- **Invalid URL during installation**: "This URL is bogus, we can't find it"
- **Authentication failure**: "These credentials are not working—we received a 401"

### The Ambiguous Error: "Unknown Error Occurred"

For most other errors, the response is vague:

```
"Unknown error occurred. Here is the integration error ID: <id>"
```

**When this happens**: The error came from the CRM system itself. Examples:

- Webhook registration failed with a CRM internal server error
- Webhook deletion failed with a CRM error
- The CRM doesn't understand the request payload we sent

### Why Ambiguous Errors Are CRM's Problem

When you receive an "unknown error":

1. **Check the error ID** in integration manager logs
2. **See what HTTP status and body** the CRM returned
3. **Escalate to the CRM team** with the specific request and response
4. The CRM developers must investigate their own configuration/code

[Erik Andersson]: > "We have a direct channel to each CRM system where we ask: 'We tried to do this request and received this unexpected response back. Could you please take a look at it?'"

### Common Uninstall Blocker: Webhook Cleanup

> "We do not allow an installation to proceed if we fail to delete webhooks because we want to be able to delete all trailing webhooks in the CRM so they don't continue to send us messages."

If webhook deletion fails with an error from the CRM, the uninstall fails and returns the ambiguous error.

### Debugging Strategy: Use Error Description to Narrow Scope

When SOC tickets come in with customer errors:

1. **Read the error description** SOC provides
2. **Determine which part of the system** had the issue:
   - Installation flow → Check integration manager
   - Uninstallation flow → Check integration manager
   - Saving mappings → Check mappings manager
   - Full sync failure → Check full sync logs
   - Real-time sync issue → Check delta sync worker
   - Outbound sync issue → Check outbound worker

[Erik Andersson]: > "You can exclude 90% of the logs just from the error description. Based on the description, you will know which log or logs you should start with."

---

## Collaborating with CRM Development Teams

### Dedicated Channels for Each CRM

The team maintains direct communication channels with each CRM's development team:

**Teams channels**:
- FCCRM and Marketing Connect Connector group with channels for:
  - E-deal
  - Efficy Enterprise
  - Tribe
  - Web CRM
  - (Others as integrated)

### Enterprise Exception: Email-Only Communication

> "The Efficy Enterprise team refuses to handle questions in Teams. They want everything in email."

**For Enterprise-specific issues**:
- Support/customer-specific problems → Must send email
- General architectural questions → Can use Teams
- Any customer-facing incident → Must escalate via email

### Escalation Process for CRM Errors

When you encounter an error from a CRM system:

1. **Review error details** (HTTP status, response body, request payload)
2. **Determine if it's customer-specific or systemic**:
   - If customer-specific: Check customer's configuration/customizations
   - If systemic: May indicate a CRM bug or our bug
3. **Open a ticket** with the relevant CRM team:
   - Include exact error response
   - Include request we sent
   - Include customer context (if specific)
4. **CRM team investigates** their system

---

## Error Classification: Error vs. Warning

### The Problem: Page Duty on Non-Actionable Errors

External system errors (CRM returning unexpected responses) currently trigger page duty alarms, but **there is nothing the on-call engineer can do in the middle of the night**:

- Can't patch code to fix CRM configuration issues
- Can't deploy to make CRM return a better response
- Can only contact the CRM team (who aren't available)

### Proposed Solution: Classify by Actionability

[Erik Andersson]: > "The judgment is: Is this something I personally can't take action on in the middle of the night? Is this something where I can deploy a patch and it's fixed? If not, it should be a warning, not an error."

### Criteria for Error vs. Warning

**Treat as ERROR only if**:
- The error indicates a bug in Apsis code you can patch
- The error is actionable during an outage

**Treat as WARNING if**:
- The error comes from external system configuration (CRM)
- The error is only solvable by the CRM team reconfiguring something
- It's something customers click (low frequency, not storm-inducing)

### Context Matters: HTTP Status + Error Message

A raw HTTP 400 is insufficient to classify:

```
400 + "Invalid campaign name length" → WARNING
  (CRM enforced a length limit, customer needs to shorten name)

400 + "Invalid request payload" → ERROR
  (We might be sending wrong data—our bug)
```

Both have the same status but different implications.

### Volume Consideration

If an error would occur **hundreds of times daily** in a synchronous service:

> "You will do nothing but sit and click acknowledge on page duty. You would definitely want to change this from an error to a warning."

### Current State

Errors are being **gradually reclassified** from error → warning as patterns emerge. Erik is open to suggestions from team members:

> "Feel free to ping me if you receive any of these [errors] and I'll check if I can change these from an error to a warning instead so you don't get interrupted by it."

---

## Installation Error Example: Campaign Name Length

### Real Incident

A customer tried to create an email campaign in Apsis with a very long name, but **Efficy Enterprise has a character limit** on campaign names in their system.

**Flow**:
1. Customer creates campaign in Apsis with long name
2. Integration tries to sync it to Enterprise
3. Enterprise rejects the request (400 error)
4. Integration returns vague "Unknown error occurred"

**Investigation required**:
1. Check integration logs for the exact CRM response
2. Identify that Enterprise enforces a name length limit
3. Contact Enterprise to confirm the limit
4. Communicate to customer: "Shorten your campaign name to X characters"

**Lesson**: This is a customer configuration/expectation issue, not a bug in Apsis.

---

## Configuration for Dynamic Consistency (Recent Fix)

### The Bug Found During Testing

During testing of a new feature (consent timeline events), Erik found that **Dynamics integrations were not bootstrapping required events and attributes** on installation.

### Root Cause

New features require updating the **generic spec configuration file**:

```
File: lib/config/connectors.go
Purpose: Sets up the inventory of all available connectors
Responsibility: Specifies which events and attributes are bootstrapped on installation
```

For each connector (e.g., Dynamics), there is a generic spec file:

```
File: lib/config/dynamics/generic_spec.go
Contains: Event definitions, attribute definitions for bootstrapping
```

**What was missing**: The generic spec file for Dynamics was created with new consent event definitions, but it **was not loaded/registered** in the connector configuration.

### Impact

When Dynamics was installed:
- The consent event was never created in the profile schema
- Any attempt to sync consent data failed
- Customer's consent updates were blocked

### The Fix

Updated `lib/config/connectors.go` to load the Dynamics generic spec:

```go
// Pseudo-code
connectors["dynamics"] = DynamicsConnector{
    genericSpec: LoadGenericSpec("lib/config/dynamics/generic_spec.go")
}
```

### Lesson Learned

For every new feature in integration:
1. Define events/attributes in the connector's `generic_spec.go`
2. **Ensure the spec is loaded** in the connector configuration
3. Test on beta before releasing to production
4. When deployed, the configuration ensures events are bootstrapped on next installation

---

## Integration Statistics Service: Planned Feature

### User Need

Product and professional services frequently request:
- "How many customers have installed this integration?"
- "How many customers are using X integration?"
- "Show me all accounts/sections where integration Y is installed"
- "What integrations is account Z using?"

**Current workaround**: Manual query of the integration database, export to spreadsheet, manual permutation of data.

### Proposed Solution

A simple **integration statistics service** to answer these queries programmatically.

### Architecture

**Data source**: Integration database (contains all installations)

**Endpoint location**: Integration Manager service

**Frontend access**: New tab in Back Office that calls the integration endpoint

**Endpoint responsibility**:
- Query integration database
- Return all installations with account/section/integration details
- Allow filtering/aggregation by Back Office

### Why This Feature Is Valuable for Team Learning

[Erik Andersson]: > "This is a simple implementation but it will touch upon every required component. It will force you to see how to add a new endpoint in integration."

The implementation path touches every part of the integration architecture:

1. **Define OpenAPI specification** for the new endpoint
2. **Generate server interfaces** from the OpenAPI spec (regenerate in all services)
3. **Implement the endpoint function** in Integration Manager
4. **Query the database** and return results
5. **Back Office frontend** integration (separate team)

This single feature exercises:
- OpenAPI specification-first development
- Code generation from specs
- Interface implementation
- Database queries
- API contract definition

### Why Not Start Immediately

- Product priority not yet confirmed
- Planning meeting scheduled for the following week
- Epic not yet created (Erik conceived the idea the evening before)
- Competes with other priorities (MA, email, SMS, pre-filled forms)

### Timeline and Commitment

- **Scope**: 2-3 days working together (vs. 1/2 day if Erik did solo)
- **Goal**: Proper walkthrough of every step, not speed
- **Start date**: Likely 2 weeks out, pending product prioritization
- **Prerequisite**: Agreement from product and Daniel (likely decision-maker)

---

## Debugging Example: SQS Message Age and Exponential Backoff

### Real Scenario: Old Queue Messages

During a review of alarms, Tomasz noticed the **outbound queue had messages that had been there for hours**.

### Why This Happens

When the CRM system returns errors, the integration implements **exponential backoff**:

```
Attempt 1: 2 seconds
Attempt 2: 1 minute
Attempt 3: 5 minutes
Attempt 4: 15 minutes
Attempt 5: 1 hour
Attempt 6: 3 hours
Attempt 7: 5-6 hours
```

### Multiplication Effect with Multiple Failures

If a customer's CRM is malfunctioning and returning errors for all 1,000 messages in a queue:

```
Message 1: Fails → Gets backoff of 12 hours
Message 2: Waits for message 1 to be processed (FIFO)
Message 3: Waits for message 2...
Message 998: Waits for 997 messages before it...
Message 1000: Effectively 12+ hour delay minimum
```

Each message waits for all prior messages to process. If each has a 12-hour backoff, the nth message might never process.

### Diagnosis Process

1. **Alarm triggers**: Message has been in queue > 1 hour
2. **Check outbound worker logs** (past 2 hours) for error patterns
3. **Look for**: Same error repeated, indicating CRM misconfiguration
4. **Identify**: Which customer/CRM system is affected
5. **Action**: Contact the CRM team to fix their configuration

### Example Investigation

During the session, messages were found from a **single customer experiencing Enterprise failures**:

[Erik Andersson]: > "I ******* hate Enterprise. This is the problem with these CRM systems—you can configure absolutely everything and everyone however way you want, and then they add custom restrictions or custom tables because essentially every customer's CRM system can become its own integration."

**Root cause**: Customer-specific Enterprise configuration issue (not Apsis bug, not standard Enterprise behavior)

**Resolution**: Email to Enterprise team to investigate the specific customer's configuration

---

## Log Group Navigation and Cloud Ops

### Log Group Naming Convention

Log group names are **directly descriptive** of their function:

```
Integration Manager        → integration-manager logs
Mappings Manager           → mappings-manager logs
Full Sync Producer         → fullsync-producer logs
Full Sync Worker           → fullsync-worker logs
Delta Sync Worker          → delta-sync-worker logs
Outbound Worker            → outbound-worker logs
```

**Finding the right logs**: Read the error description from SOC and determine which service handled that part of the flow.

### Cleanup in Progress

Erik noted several legacy artifacts that will be cleaned up to reduce confusion:

**Legacy items to be deleted**:
- Ping-pong lambdas (old implementations)
- Old Lambda implementations not currently used
- ~10 pages of SQS queues from previous Full Sync architecture development (created during testing, never torn down)

**When cleaned**: Before the new team members need to navigate these logs

### Auto Scaling False Positives

> "Auto scaling alarms get triggered by default for some reason. If you see 6 alarms from auto scaling, just hide them—it's a false positive."

If you see auto scaling alarms in CloudWatch, they can usually be dismissed without investigation.

---

## Summary: Incident Prevention and Recovery Strategies

### Design Principles

1. **Recovery over prevention**: Integration is resilient because data can always be recovered, not because incidents can't happen
2. **Multiple copies**: Data exists in CRM and Apsis; both can be sources of truth
3. **Idempotent operations**: Full sync can be re-run to restore state
4. **Graceful degradation**: Single customer issues don't block other customers (after message grouping fix)

### Monitoring Strategy

- **Don't alert on volume**: Alert on message age instead
- **Understand FIFO limitations**: AWS FIFO has architectural boundaries
- **Classify errors properly**: Distinguish between Apsis bugs and CRM configuration issues
- **Actionability test**: Only page duty on errors an engineer can fix

### Collaboration Model

- Direct channels to each CRM development team
- Escalation path for customer-specific issues
- Enterprise team has special email-only requirement
- Regular sync meetings to discuss integration changes

### Common Incident Patterns

1. **FIFO queue exhaustion** → Monitor message age, check for backoffs
2. **Database migration miss** → Verify migrations before release, fail fast
3. **Third-party API changes** → May require HTTP client fixes (rare but critical)
4. **Customer configuration issues** → Escalate to CRM team with context
5. **Installation/uninstallation blocks** → Check integration manager logs, may need CRM investigation

---

## Key Takeaways

1. **Integration is resilient by design** because data exists in multiple places (CRM and Apsis) and full sync can restore any state. True data loss is nearly impossible.

2. **SQS FIFO queue limitations** are the most common recurring incident. AWS only checks the first 20,000 messages for group identification. Mitigated by including CRM ID in message group keys. Monitor message age (not count) to detect stalls.

3. **Database migrations are manual and error-prone**. Always check `delta.sql` diffs before merging and apply migrations before releases. Consider building automation, but none found yet in Golang ecosystem.

4. **The MA catastrophic incident** (400,000+ stuck jobs for 2.5 weeks) was caused by an HTTP client that hung on empty response bodies when receiving 429 status codes. Detection failed because no logs were generated (request hung). Led to implementation of message age monitoring via Lambda.

5. **Installation/uninstallation errors** are usually CRM system issues, not Apsis bugs. Ambiguous "unknown error" messages mean the CRM returned an error we don't control. Always escalate to the CRM team with the specific response and request details.

6. **Error vs. warning classification** should be based on actionability: Page duty only on errors you can patch at 3 AM. External system configuration issues should be warnings so you can notify the CRM team without interrupting sleep.

7. **CRM development collaboration** happens through dedicated Teams channels (except Enterprise, which requires email). Each CRM system is effectively its own integration due to customizations and configuration flexibility.

8. **Log group discovery** is straightforward—group names describe their function. Use error descriptions from SOC to determine which logs to check, and you'll quickly learn the patterns.

9. **The integration statistics service** is a valuable learning project that touches every part of the integration architecture (OpenAPI spec, code generation, endpoint implementation, database queries). Scheduled for prioritization in the next week.

10. **Full sync is the ultimate recovery mechanism** but is expensive for customer CRMs, so use judiciously. Can be triggered internally or requested from customers.

---

## Unresolved Questions and Action Items

### Action Items

1. **Erik**: Create email to Efficy Enterprise team about customer's configuration issue causing outbound queue delays
2. **Erik**: Invite Michal and Tomasz to the FCCRM and Marketing Connect Connector Teams channels for ongoing CRM collaboration (planned for January onboarding)
3. **Erik**: Delete legacy log artifacts (ping-pong lambdas, old queues from Full Sync testing) to reduce confusion
4. **Erik**: Complete deployment of recent configuration fix for Dynamics consent events to production
5. **Product + Team**: Schedule meeting to prioritize the integration statistics service feature
6. **Team**: If the statistics service is approved, plan 2-3 day workshop starting in ~2 weeks

### Open Design Decisions

1. **Database migration automation**: Should this be automated in Go? Possible paths: custom bash scripts, or newer migration framework adoption. No decision made yet.

2. **Exponential backoff strategy**: The current backoff is aggressive (up to 12+ hours). Discussion ongoing about whether to adjust this strategy for malfunctioning CRM systems.

3. **Error classification**: Gradual migration from error → warning for external system issues. Criteria being developed and refined based on incidents.

### Deferred Learning

- Detailed walkthrough of CRM development team collaboration will happen as Michal and Tomasz handle support cases in January
- Full architecture of Back Office integration for the statistics service not yet designed
- Custom bash or framework solution for database migrations not yet explored