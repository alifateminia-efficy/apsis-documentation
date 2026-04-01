---
source_file: Erik - Incidents in Integrations.txt
domain: Apsis One Integrations
topics: [incident management, SQS FIFO queue limitations, database migrations, MA job processing outage, installation/uninstallation errors, PagerDuty alarm tuning, CRM error handling, outbound worker backoff strategy, integration statistics service, OpenAPI spec-first development]
speakers: [Erik Andersson (senior/lead engineer, integration domain), Michal Rosikiewicz (incoming team member), Tomasz Kowalski (incoming team member), Lukasz Grabowski (transcription initiator)]
key_components: [SQS FIFO queues, Delta Sync queue, Outbound Worker, Integration Manager, Mappings Manager, Full Sync producer, BullMQ, MA (Marketing Automation), Apsis One, Dynamics connector, FCC CRM systems (Enterprise, Tribe, eDeal, WebCRM), PagerDuty, CloudFormation, OpenAPI/Swagger]
session_type: knowledge-transfer
---

# Incidents in Integrations — KT Session

## Session Overview

Erik Andersson leads a knowledge transfer session for incoming team members Michal and Tomasz covering the most common and severe incidents in the Integrations domain. The session covers three major incident categories: SQS FIFO queue flooding, missing database migrations, and a severe MA job-processing outage caused by an empty HTTP response body. The session also covers real-time PagerDuty alarm investigation (triggered by live incidents during the call), CRM error handling philosophy, alarm severity tuning, and a proposed upcoming "integration statistics" feature to serve as a hands-on learning project. The session ends with Erik committing to email the FCC Enterprise CRM team about active errors found during the live review.

---

## Data Loss Risk Profile in Integration

Erik's opening framing is important context for how the team thinks about severity:

> "Data loss is the worst case, but by nature it is quite hard to encounter real data loss in integration."

The key recovery mechanism is the **full sync**: it downloads every contact, all mapped data, and all consent data from the CRM. This means:

- If a delta sync is accidentally misconfigured and real-time messages are discarded, the data still exists in the CRM.
- If the CRM has a bug and fails to process consents sent from Apsis, Apsis holds the correct consent state and can re-export it to the CRM.
- The system is designed so the correct state always exists in *either* Apsis *or* the CRM, making recovery possible.

**Caveat:** Full syncs put significant stress on the customer's CRM system, so they should not be triggered too frequently. Internal triggering is possible but should be used judiciously.

---

## Incident Type 1: SQS FIFO Queue Flooding

### The AWS FIFO 20,000-Message Limit

**AWS SQS FIFO queues** only inspect the first 20,000 messages when evaluating message groups. This is a hard AWS implementation limitation with serious operational consequences.

**How message grouping works in Integration:**
- Messages are grouped by: `account_section + integration_id + CRM_id`
- This ensures ordering is preserved per contact (e.g., update #1 must be processed before update #2 for the same contact)

### The Original Design Flaw and How It Caused Incidents

The *original* design grouped messages only by `account_section + integration_id` (without the CRM ID). This meant that when a backoff was placed on a message group, it applied to *all* messages from that customer's integration.

**Failure scenario:**
1. A customer performs a mass CRM schema change (e.g., adds a field to every contact), generating ~200,000 webhook update messages.
2. One message fails and a backoff is applied to the message group.
3. Queue depth reaches ~120,000 messages.
4. AWS only scans the first 20,000 messages for group IDs, so it only sees the backed-off group.
5. **All messages from every other customer are blocked** until the queue drains below 20,000.

**Mitigation used at the time:**
- Blacklist the offending installation in the consumer — discard all messages from that installation ID on receipt.
- Notify SOC that messages were discarded and the customer needs to run a full sync.
- Once the queue drains, consumption resumes normally.

### Current State (Post-Fix)

The message group now includes the CRM ID (`account_section + integration_id + CRM_id`). This means a backoff only applies to a specific contact's messages, not the entire customer's integration. This significantly reduces blast radius.

### The Two Queues Most Vulnerable to Burst Flooding

**1. Delta Sync queue (inbound, real-time messages)**
- Customers can trigger huge bursts by doing database migrations (e.g., adding a field to every contact) without disabling webhooks first.
- Best practice to communicate to customers: **disable webhooks before performing CRM database migrations**.

**2. Outbound Worker queue**
- Email activity sendings can generate very large message volumes: sent + delivered + opened + clicked events × number of recipients.
- Example: Hula Press in Tribe can produce approximately **500,000 events** in this queue from a single send.
- Grouping in the outbound worker helps handle this, but the queue remains a watch point.

### Alarms

An alarm is in place that triggers when messages have been sitting in the queue **longer than approximately one hour** (age-based, not count-based — see also the MA section for why count-based alarms are inappropriate).

**When the alarm fires, check:**
1. How many messages are currently in the queue?
2. Are there errors in the consumer logs?
3. Is there a backoff applied to essentially all message groups, or is something specifically stuck?

> **Note:** This FIFO 20,000-message limitation applies to *any* service using SQS FIFO queues, not just Integration. Any team using FIFO queues should be aware of this.

---

## Incident Type 2: Missing Database Migrations

### Migration Approach in Integration

Integration does **not** use automatic database migrations. There are two SQL files:

- **Schema create file** — full database structure, used only when setting up from scratch.
- **Delta SQL file** — iterative updates. When a feature adds a new table or column, the SQL command is added here.

### The Required Manual Process

Before merging to `product` or `master` branches, check whether the delta file has changed. If it has:
1. Set up a database connection.
2. Manually run the corresponding SQL commands from the delta file against the target environment's database.

[Erik Andersson]: *"I very much encourage you to see if you can find any cool way of doing this automatically. We've failed so far to find any nice automatic way to do this in Golang, but it might be possible with some custom bash script or if there has been any new framework released."*

### Failure Mode and Recovery

If a migration is missed and a service starts:
- The service will fail with a database error (missing column/table).
- This will trigger a PagerDuty alarm quickly — you will notice fast.
- Messages in the queue have retries, so once the migration is applied, the next retry will succeed.
- **Worst case:** customers may need to run a full sync if real-time sync messages were affected during the outage window.

**Assessment:** Annoying but recoverable. Not a severe incident. The risk is catching it *before* release rather than after.

---

## Incident Type 3: The MA Job Processing Outage (Most Severe Ever)

This is described by Erik as *"by far the most serious incident we've ever had to handle in our team."*

### Root Cause

1. The **Audience** system changed its handling of **429 (rate limit) responses**: they removed the response body and returned only the HTTP 429 status code.
2. A shared HTTP client library used in **MA** had a bug: if it received a non-2xx response *with an empty body*, the request never terminated — it hung indefinitely and never returned a response to the caller.
3. MA has a pool of worker/consumer processes. One by one, each worker encountered this hanging request. Eventually, **all consumers for the MA service were stuck** in this hung HTTP request.

### Why It Wasn't Detected

Because the hung function never returned, it produced **neither success logs nor error logs**. No PagerDuty alarm was triggered. The issue was discovered manually after approximately **2.5 weeks**.

### Impact

- Over **400,000 jobs in MA were stuck** and unprocessed.
- MA is **highly time-sensitive** — unlike Integration, where delayed processing is usually recoverable, MA flows are triggered by events (e.g., "user purchased → send discount") with hard time windows.
- Example: customers received promised discounts 2.5 weeks late, purchase confirmations were never sent, flows were broken.
- **You cannot simply rerun the jobs.** Each customer had to be contacted individually for each affected flow to determine whether a retry was appropriate or whether the business context had expired.

### Remediation Effort

[Erik Andersson]: *"Me and Shervidia sat until 9:00 for like 1.5 weeks just to get MA back to the state it should be."*

### Post-Incident Changes: Age-Based Queue Monitoring Lambda

A **Lambda function** was implemented that continuously monitors the BullMQ job queues in MA and checks the **age of the oldest message**, not the count. This is important:

- A count-based alarm would fire every time a large export is triggered (e.g., 200,000 messages from a legitimate bulk operation). This would be a constant false positive.
- An **age-based alarm** correctly distinguishes between "queue is busy processing" and "queue is stuck."

[Michal Rosikiewicz]: *"That's obvious because we cannot predict from a big fan [large job]."*
[Erik Andersson]: *"Exactly. What you want to check is: how old is the message in the queue? How long has it been stuck there? We needed a custom function to check that."*

---

## Common Support Request: Installation and Uninstallation Errors

These are the most frequent inbound questions from professional services and SOC.

### Explicit Error Messages (Handled Gracefully)

- **Incorrect URL on install:** Error message explicitly states the URL is invalid.
- **Auth failure on install:** Error message states credentials are not working (HTTP 401).

### Ambiguous "Unknown Error" Cases

For any other error — including errors during webhook registration or webhook deletion — the system returns a generic:
> `"An unknown error occurred. Here is the integration error ID."`

This is typically because the CRM returned an unexpected error (e.g., internal server error, permission misconfiguration) that Integration cannot interpret.

**Most common uninstall error:** Failure to delete webhooks from the CRM. The current behavior is to **block the uninstallation** if webhook deletion fails, because trailing webhooks in the CRM would continue sending data to Apsis even after the integration is removed.

### Triage Approach for Support Tickets

Start by identifying which phase the problem occurred in, then go to the corresponding log group:

| Problem Description | Log Group to Check |
|---|---|
| Installation / uninstallation failure | Integration Manager logs |
| Saving mappings failed | Mappings Manager logs |
| Full sync failed | Full Sync producer logs |
| Real-time sync issues (inbound) | Delta Sync worker logs |
| Outbound activity sync issues | Outbound Worker logs |

[Erik Andersson]: *"Based on the description of the problem, you can exclude 90% of the logs immediately. Is the problem in the inbound flow, the outbound flow, or in setting up the integration? You'll get the hang of this after a couple of support cases."*

SOC tickets are often thorough and may include a direct link to the relevant log entry. Always check the ticket description carefully before starting a log search.

### Deleting a Stuck Integration

If a customer cannot uninstall via the UI, there is a **`clear integration` endpoint** that can be called with specific flags to force-delete the installation. The recommended method is to use the **Postman collection** maintained for the integration service rather than constructing the `curl` command manually.

---

## PagerDuty Alarm Philosophy and CRM Error Handling

### Live Incident Review During Session

Two PagerDuty alarms were active during the session (shared by Tomasz). Both were `external_system_error` errors from **FCC Enterprise** — the Integration service attempted to create or update an email campaign and received an unexpected error response.

### Error Severity Assessment Framework

[Erik Andersson]:
> "The combination of the HTTP status and the error message makes me decide: should this be an error or a warning? You cannot just say 'it's a 400, so it's a warning.' A 400 can also mean we sent a malformed request payload — that *is* something you can deploy a patch for. But if the combination of status and error message indicates a CRM-side configuration issue, then there is nothing deployable in Apsis that fixes it."

**Decision criteria for error vs. warning:**
- Can I deploy a patch in Apsis to fix this right now? → **Error** (actionable)
- Is this a CRM system configuration issue that requires the CRM team to investigate? → **Warning** (not actionable at 3am)

Erik has been progressively changing external CRM errors from `ERROR` to `WARN` in the logging, especially for synchronous/high-frequency paths where these errors could cause constant PagerDuty noise.

**Important caveat on 400 errors specifically:** A 400 from a CRM *could* indicate a bug on Apsis's side (malformed payload). Always check the specific error message, not just the status code.

### Communicating with CRM Teams

Integration maintains **dedicated Microsoft Teams channels** for each CRM partner:
- eDeal
- Enterprise
- Tribe
- WebCRM

**Exception — FCC Enterprise:** The Enterprise team refuses to handle support/incident questions in Teams. All error investigations and specific customer issue reports must be sent via **email**. General architectural/planning questions can go in Teams, but anything support-related requires email.

[Erik Andersson]: *"For any enterprise questions about 'we received this error for this specific customer,' they want that in an email. So I'm going to send them an email about it."*

Erik committed to adding Michal and Tomasz to the CRM team channels (Lukasz had been added the day before) and to CC-ing them on the email to the Enterprise team.

### Exponential Backoff in Outbound Worker

The outbound worker uses an exponential backoff strategy on failed messages:
- Progression: seconds → ~1 min → ~5 min → ~15 min → 1 hour → 3 hours → ~5–6 hours

**Compounding problem with a malfunctioning CRM:**
If the CRM is returning errors for all messages from a customer, and the customer has 1,000 messages queued, each message accumulates its own backoff. By the time the 998th message is reached, the backoff on earlier messages may be 12+ hours. The practical effect is that recovery for a large backlog with a broken CRM can take an extraordinarily long time. This backoff strategy has been discussed internally but not changed.

[Erik Andersson]: *"SQS also won't show you messages that are in a visibility timeout when you pull from the queue — so you may not even see them when inspecting the queue depth."*

---

## Log Groups and CloudWatch Housekeeping

Log group naming convention follows service names and is generally self-describing:
```
integration-manager
mappings-manager
fullsync-producer
delta-sync-worker
outbound-worker
```

### Known Clutter to Be Cleaned Up

Erik noted several sources of noise in the AWS environment that he intends to clean up to avoid confusion for new team members:

1. **Legacy "ping pong" Lambdas** — old Lambda implementations for older services, no longer used.
2. **Old Lambda implementations** from pre-current architecture, still present in CloudWatch but not active.
3. **Stale SQS queues in staging** — during the development of the new full sync architecture, queues were spun up per full sync job and not fully torn down. Staging currently has approximately 10 pages of leftover SQS queues.

### Auto-Scaling Alarm False Positives

CloudWatch auto-scaling alarms are apparently in an alarm state by default for unknown reasons. These are **false positives**.

> **Action:** When reviewing CloudWatch alarms, hide auto-scaling alarms to avoid confusion. They do not indicate a real problem.

---

## Live Bug Discovery: Dynamics Connector — Generic Spec Not Loaded on Installation

### What Happened

During post-session testing of a recent feature release (consent timeline events), Erik discovered a breaking bug in the **Dynamics connector** configuration.

### Root Cause

The file `generic_spec.go` defines events and attribute definitions that should be bootstrapped when a connector is installed. It contains consent-related event definitions shared across connectors.

The connector inventory is configured in:
```
lib/config/connectors
```
(referred to as the "lib config connector file" — this file registers all available integrations on service startup, including any custom attributes or events to bootstrap on installation)

Dynamics had never previously required any custom bootstrapped attributes, so it had no entry in this config. When the new `generic_spec.go` was added for Dynamics as part of the consent timeline feature, it was added to the file but **not registered in the connector config**. Result: installing Dynamics would never load the generic spec events, so consent events were never bootstrapped.

**Observed failure:** All consent updates failed because the consent event type did not exist in the installation.

### Resolution

- Fix merged to beta branch, propagated, and production release was being prepared during the session.
- No further testing needed for this specific fix.

---

## Upcoming Feature: Integration Statistics Service

### Background and Motivation

Product frequently requests data such as:
- How many customers have installed a specific integration?
- Which sections have a given integration installed?
- What is the full list of installations across accounts?

Currently this data is only available by querying the **integration database** directly and manually exporting/permutating the results.

### Proposed Design

- Add a new **endpoint to the Integration Manager** that returns installation statistics.
- **Back Office** calls this endpoint to display the data in a new tab.
- This will be discussed with product the following week for prioritization.

### Why This Is a Good Onboarding Feature

[Erik Andersson]: *"It will touch every required component. It will force you to see how to add a new endpoint in Integration, which will touch our OpenAPI specification, require regenerating the server interfaces in all services, and then implementing the corresponding function on the Integration Manager server."*

Timeline estimate: Erik could implement it alone in half a day; doing it together as a learning exercise would take 2–3 days with proper walkthrough of every step.

---

## OpenAPI Spec-First Development in Integration

Integration uses a **spec-first** approach for API development:

1. Define the endpoint in the **OpenAPI specification** first.
2. Generate server interfaces from the spec (auto-generated code — do not edit manually).
3. Implement the actual handler function against the generated interface in the Integration Manager.

**Why this approach was adopted:**
Previously, endpoints were defined in CloudFormation files, and separately the OpenAPI spec was maintained for documentation and sharing with other teams. These two definitions easily got out of sync.

> "Now when we take this approach where we define it first, that forces us to always have an up-to-date API specification and also it reduces clutter in the CloudFormation files."

This results in an always-current Swagger doc that can be shared with other teams.

---

## Key Takeaways

1. **Integration is designed for recoverability.** The full sync is the primary recovery tool — correct state always exists in either Apsis or the CRM. Don't hesitate to ask customers to run a full sync, but be mindful of the load it places on their CRM.

2. **SQS FIFO queues have a hard 20,000-message scan limit for message groups.** This can cause all messages from other customers to be blocked behind a flooded customer's messages. The current message grouping (including CRM ID) mitigates this significantly, but the Delta Sync queue and Outbound Worker queue remain the two key watch points.

3. **Database migrations in Integration are manual.** Always check the delta SQL file for changes before merging to `product` or `master`. Run migrations against the target database before deploying.

4. **The MA hung-request outage is the team's most severe incident.** 2.5 weeks of undetected job processing failure resulted from a hung HTTP client that produced no logs. Age-based queue monitoring (not count-based) is the correct detection approach for job queues.

5. **Most PagerDuty alerts in Integration are caused by CRM-side errors**, not Apsis-side bugs. Use the combination of HTTP status code + error message to assess whether the error is actionable. Many should be `WARN` rather than `ERROR`.

6. **Enterprise CRM team requires email for all customer-specific issues** — not Teams. Other CRM partners (Tribe, eDeal, WebCRM) can be reached via dedicated Teams channels.

7. **Start log investigation by identifying the flow phase** (installation, inbound sync, outbound sync, mapping) to immediately narrow to the correct log group.

8. **Auto-scaling CloudWatch alarms are a known false positive** — hide them when reviewing alarms.

9. **Integration uses OpenAPI spec-first development.** Always define the endpoint in the OpenAPI spec before implementing, then regenerate server interfaces.

10. **Exponential backoff on failed CRM calls compounds severely** for large backlogs with a broken CRM. A single malfunctioning installation can effectively be "queued" for 12+ hours per message.

---

## Unresolved Questions and Action Items

- [ ] **Erik:** Send email to FCC Enterprise team regarding the active `external_system_error` for the specific customer's campaign; CC Michal and Tomasz.
- [ ] **Erik:** Add Michal and Tomasz to the CRM team channels in Microsoft Teams (Lukasz already added).
- [ ] **Erik:** Delete legacy CloudWatch log groups (ping pong Lambdas, old Lambda implementations) to reduce confusion.
- [ ] **Erik:** Delete stale SQS queues in staging from the old full sync architecture development.
- [ ] **Erik / Team:** Discuss the Integration Statistics Service with product and Daniel the following week to get prioritization and agree on timing (target: ~2 weeks out).
- [ ] **Team:** Investigate whether automatic database migrations are feasible in Golang (bash scripts or a new framework) — currently all manual.
- [ ] **Open question (⚠️ ambiguous):** The outbound worker exponential backoff strategy (seconds → hours) has been discussed multiple times internally but no decision to change it was captured. It is unclear whether there is a formal plan to revisit this or whether the current approach is considered acceptable.
- [ ] **Open question:** The exact name of the shared HTTP client library in MA that caused the hung-request incident was not recalled during the session. This library should be identified and confirmed as patched.