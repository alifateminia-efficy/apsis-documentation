---
source_file: Erik - Reduce AWS cost Integrations.txt
domain: Apsis One Integrations
topics: [AWS Cost Optimization, CloudWatch Logging, RDS Database Optimization, Sync Performance, Large Customer Handling]
speakers: [Erik Andersson, Speaker 1 (Grzegorz Kozub), Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [CloudWatch, RDS Aurora Serverless, ECS Tasks, Sync Conditions, CRM Integration, API Credentials Caching, Events Definition IDs]
session_type: cost-optimization-session
subdomains: [Architecture]
---

## Session Overview

This session focused on identifying quick wins for reducing AWS costs in the Apsis One Integrations domain. The domain's integration costs are approximately $1.3K USD monthly, representing roughly 1/120th of the total Apsis platform costs. The team identified two primary cost drivers—CloudWatch logs and RDS database—and discussed both immediate optimization opportunities (log cleanup, database scaling) and longer-term architectural changes (pushing sync conditions to the CRM side rather than processing client-side).

---

## Current Cost Baseline and Priorities

**Monthly integration costs:** $1.3K USD

The domain is already considered well-optimized from a cost perspective. The two highest-cost services identified through AWS Cost Explorer are:
1. **RDS (Relational Database Service)**
2. **CloudWatch (logging and monitoring)**

All other service costs are not substantial enough to warrant optimization effort at this time.

---

## CloudWatch Logging Optimization

### Essential Logging vs. Debug Logging

[Erik Andersson]: There is a distinction between tracing that **must remain** for operational purposes and debug logging that can be removed.

**Essential logging that must be preserved:**
- CRM IDs (used to trace why a specific profile didn't sync to Apsis or didn't receive consent)
- Message receipt acknowledgments (we received a message for this contact)
- Processing status (successfully processed this contact for consent, profile updates)
- Flow entry/exit events (profile entering/exiting specific flows)

**Debug logging that can be removed:**
- Status logs for configuration details
- Mapping logs
- Request debug logs
- General info logs

[Erik Andersson]: > "I would almost go so far as to say you can probably remove half of the logs we have."

### Rationale for Current Verbosity

The verbosity of current logging exists because the cost of optimization effort previously exceeded the savings. However, this analysis should be revisited if operational debugging needs have been met.

---

## RDS Database Optimization

### Aurora Serverless Configuration

**Current configuration:**
- Aurora Serverless instances with 3 ACUs (Aurora Capacity Units) as minimum

**Optimization opportunity:**
[Erik Andersson]: The minimum ACU count can be reduced from 3 to 2, or potentially to 1, because integrations should theoretically not be database-heavy.

### Caching Strategy

The integration domain should be minimally database-dependent through aggressive caching. Currently, the system retrieves data once but uses it thousands of times:

**Examples of cacheable data:**
- **API credentials for customers** - retrieved once, used 40,000 times across a sync
- **Event definition IDs or discriminators** - retrieved once, used 400,000 times

**Caching logic:**
- Cache is populated at the start of a full sync
- For real-time message processing, cache is populated on receipt
- Cache should persist in memory for approximately 10 minutes between refreshes
- This eliminates redundant database queries during a sync cycle

If caching is missing for any frequently-accessed data, it should be added immediately.

---

## The Large Customer Problem: Sync Conditions Processing

### Issue Description

Large customers with millions of contacts create disproportionate cost and performance overhead. A specific case discussed:

**Customer scenario:**
- Total contacts in CRM: 8 million
- Sync conditions: Only sync Swedish customers (based on country field)
- Contacts matching sync condition: 100,000

**Current problematic behavior:**
The sync conditions are evaluated **inside Apsis**, not pushed to the CRM system. This means:
- All 8 million contacts must be downloaded from the CRM
- All 8 million contacts must be processed locally
- Only 100,000 are ultimately synced
- 7.9 million contacts are discarded after processing

**Performance and cost impact:**
- Full sync runs for 1.5 days with significant CPU consumption
- CloudWatch logs are multiplied by the number of contacts processed, not synced
- This creates massive logging costs even though most data is discarded
- CRM systems timeout due to overload (1.5 day processing window)
- System memory is consumed processing contacts that will be discarded

[Erik Andersson]: > "For these big customers, we are processing 8 million contacts of which we sync 100,000 and this is an insane way of doing it. Like we haven't technically approved these sizes customers, but it has just happened now."

### Root Cause

Sync conditions (name starts with X, country equals Y, status equals active, etc.) are filtering rules applied within Apsis after data retrieval, not pre-filtering requests sent to the CRM system.

---

## Solution: Push Sync Conditions to CRM

### Architecture Change

Sync conditions should be sent to the CRM system **before** data retrieval. The CRM would then return only the filtered 100,000 records instead of all 8 million.

**Benefits:**
- Reduces data transfer by ~98% for this customer
- Reduces CPU time proportionally
- Reduces CloudWatch logs proportionally
- Prevents CRM timeouts and system memory exhaustion
- Makes the system scalable for large customers

[Erik Andersson]: > "We will only be presented with the 100,000 that will be included there."

### Implementation Note

This is a separate, longer-term track from the immediate log cleanup optimization. It is **necessary** for the system to handle large customers sustainably.

[Erik Andersson]: > "It's a necessary track, but that's for us to be able to handle these big customers because we can't, we can't do it otherwise like the sinks die because of memory and the CRMS time out for 1 1/2 day because they are getting overloaded."

---

## Recent Debug Logging Spike Investigation

A spike in costs coincided with a recent release. Suspected cause is the addition of new debug logging that should be removed. The exact timing and release correlation needs to be verified by reviewing deployment history around the spike date.

The large customer processing inefficiency may also contribute to this spike, but the debug logging cleanup is a separate, immediate win.

---

## Other Services Optimization Considered and Rejected

### Microservice to Single ECS Task Consolidation

[Erik Andersson]: The integration domain currently runs as separate microservices, each in its own ECS task. A potential optimization would be to combine all integration services into a single ECS task.

**Decision:** Not recommended.

**Rationale:**
- The development time required would be substantial
- The cost savings would not justify the development effort
- It would increase operational complexity
- The gains would be minimal compared to the effort involved

---

## Key Takeaways

1. **Immediate low-hanging fruit:** Remove approximately 50% of CloudWatch debug logging while preserving essential tracing information (CRM IDs, processing status, flow transitions). This is quick to implement and has clear cost savings.

2. **Database optimization:** Reduce Aurora Serverless minimum ACUs from 3 to 2 (or potentially 1) and verify that caching is comprehensive for all frequently-accessed data (API credentials, event definitions, discriminators).

3. **Large customer sustainability:** Pushing sync conditions to the CRM side is essential for handling existing large customers and preventing system timeouts/memory exhaustion. This is a separate track from the immediate optimizations.

4. **Cost baseline:** The integration domain is already well-optimized; other services (beyond CloudWatch and RDS) have negligible cost impact and should not be prioritized for optimization.

5. **Debug logging audit:** Investigate and remove debug logs added in recent releases that may have contributed to the cost spike.

---

## Unresolved Questions and Follow-up Items

- [ ] Exact correlation between the French customer onboarding date and the CloudWatch cost spike needs to be verified
- [ ] Review release notes/deployment history around the spike date to identify newly-added debug logging
- [ ] Verify that all cacheable data patterns (API credentials, event definitions, discriminators) are properly cached with appropriate TTLs
- [ ] Design and schedule the implementation of pushing sync conditions to the CRM system
- [ ] Test Aurora Serverless scaling behavior when minimum ACUs are reduced from 3 to 2 or 1