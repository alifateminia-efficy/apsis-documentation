---
source_file: Erik - Reduce AWS cost Integrations.txt
domain: Apsis One - Integrations
topics: [AWS Cost Optimization, CloudWatch Logging, RDS Database Tuning, Sync Conditions Performance, Full Sync Processing, Caching Strategy, Large Customer Handling]
speakers: [Speaker 1 (likely Grzegorz Kozub - session lead), Erik Andersson (Integrations domain expert), Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [CloudWatch, RDS Aurora Serverless, ECS Tasks, CRM Integration, Sync Engine, API Credentials Caching, Event Definitions]
session_type: knowledge-transfer
---

## Session Overview

This session focused on identifying low-hanging fruit cost reduction opportunities in the Apsis One Integrations domain, which currently costs approximately $1,300/month in AWS expenses. The primary cost drivers identified are CloudWatch logging and RDS database resources. Erik Andersson presented concrete recommendations: reducing debug logs by approximately 50%, optimizing Aurora serverless ACU minimums from 3 to 1-2, and implementing aggressive caching strategies. A significant architectural issue was discussed regarding handling large customers (8M+ contacts) where sync conditions are evaluated client-side rather than server-side, causing massive resource waste and log inflation.

---

## Current Cost Structure and Scope

The Integrations domain represents a relatively small portion of overall platform costs ($1,300/month, approximately 1/120th of total platform costs). The team acknowledged that this domain is already fairly optimized, so the focus is on identifying remaining low-hanging fruit without deep architectural changes.

**Primary cost drivers:**
- RDS (database)
- CloudWatch (logging)

Other services do not warrant optimization effort at this time.

---

## CloudWatch Logging Optimization

### Distinguishing Critical Tracing from Debug Logs

[Erik Andersson]: The key principle here is distinguishing what must remain for tracing versus what can be safely removed.

**Must-keep tracing elements:**
- CRM IDs (required to answer critical questions like "why didn't this profile sync?" or "why didn't consent get updated?")
- Contact receipt messages and processing status
- Flow entry and exit events
- Successful processing confirmations for consent and profile updates

**Can-be-removed debug logs:**
- Info-level logs for status tracking
- Configuration and mapping logs
- Debug logs for request processing

[Erik Andersson]: > "I would almost go so far as to say you can probably remove half of the logs we have already."

### Why Debug Logs Accumulated

The logs accumulated because, historically, the cost of optimization time exceeded the cost savings. Now that the issue is being addressed proactively, there's an opportunity to clean them up.

### Impact of Large Customers on Log Volume

The logging problem is significantly exacerbated by large customers processing massive datasets. When a customer has 8 million contacts but filters for only 100,000 (e.g., Swedish customers only), the system still logs entries for all 8 million processed contacts before discarding 98% of them. This multiplier effect makes CloudWatch costs disproportionately high.

---

## RDS Database Optimization

### Aurora Serverless Configuration

The current configuration uses **Aurora serverless with a minimum of 3 ACUs (Aurora Capacity Units)**. 

[Erik Andersson]: Recommendation is to investigate reducing this to 2 ACUs or potentially even 1 ACU, with the rationale that:
- The Integrations domain should not be database-heavy in theory
- Most data should be cached rather than fetched from the database repeatedly

### Caching Strategy

[Erik Andersson]: The integration domain should implement aggressive in-memory caching to avoid unnecessary database queries:

> "We need to retrieve like API credentials for a customer and then use that like 40,000 times. We need to get events definition IDs or discriminators or whatever and then use that 400,000 times."

**Caching principles:**
- API credentials should be fetched once per cache cycle and reused thousands of times
- Event definition IDs and discriminators should remain in memory across requests
- Cache TTL should be approximately 10 minutes
- At the start of a full sync, data must be fetched fresh, but during subsequent requests within the cache window, in-memory copies should be used

[Erik Andersson]: If caching is not already implemented for frequently-accessed items, it must be added. This is a necessary optimization to keep database load low.

---

## Architectural Issue: Large Customer Sync Conditions

### The Problem

A critical architectural bottleneck exists in how **sync conditions** are evaluated. Currently:

1. Sync conditions are evaluated **inside Apsis** (client-side evaluation), not in the CRM system
2. The CRM has no knowledge of the filtering criteria
3. When a customer wants to sync only Swedish contacts from 8 million total contacts, the system must:
   - Download all 8 million contacts
   - Process all 8 million contacts locally
   - Filter to 100,000 that match criteria
   - Discard 7.9 million

This results in:
- Massive CPU wastage during processing
- Exponential log volume multiplication (logs are generated per contact processed, not per contact synced)
- Full sync cycles running for 1.5+ days with high CPU consumption
- CRM timeouts when overloaded by excessive download requests

[Erik Andersson]: > "For these big customers, we are processing 8 million contacts of which we sink 100,000 and this is an insane way of doing it. Like we haven't technically approved these sizes. Customers, but it has just happened now."

### Planned Solution

Sync conditions will be **pushed to the CRM system** so that:
- The CRM applies filters before returning data
- Only the matching 100,000 contacts are downloaded
- Processing and logging only occur for relevant contacts
- Reduction in scale would be "an insane amount" according to Erik

[Erik Andersson]: > "So the sync conditions will be sent to the CRM system. Instead of us downloading 8 million contacts, we will only be presented with the 100,000 that will be included there."

This is a separate optimization track from the logging cleanup but is essential for handling large customers.

### Recent Unexplained Cost Spike

There was a recent cost spike that may correlate with onboarding a large French customer. [Erik Andersson] suspects additional debug logging was added around the time of release, which should be removed. The exact correlation between the customer onboarding and cost increase was not definitively established, but it is considered likely that large customers contribute significantly to CloudWatch costs during full sync operations.

---

## Other Architectural Considerations

### Microservice Consolidation

[Erik Andersson] noted that every service in the Integrations domain runs as a separate ECS task. Consolidating them into a single task could theoretically reduce costs, but:
- The development effort would be substantial
- The actual cost savings would be minimal
- The complexity added would outweigh the gains
- This is not recommended as a priority

---

## Key Takeaways

1. **Quick wins exist in CloudWatch logging**: Remove approximately 50% of current debug logs while preserving critical tracing (CRM IDs, processing status, flow events). This should be implemented immediately.

2. **RDS optimization opportunity**: Investigate reducing Aurora serverless minimum from 3 ACUs to 1-2 ACUs in conjunction with improving caching implementation.

3. **Caching must be aggressive**: API credentials, event definitions, and discriminators should be cached in memory for ~10 minute intervals. If not already implemented, this is a critical missing piece.

4. **Large customer scaling is an architectural problem**: The current approach of evaluating sync conditions client-side is unsustainable for customers with millions of contacts. Moving sync condition evaluation to the CRM system is necessary but is a separate track from immediate cost reduction.

5. **No other services warrant optimization**: Other AWS services do not have costs significant enough to justify optimization effort at this time.

6. **Development ROI**: Any optimization requiring significant development time (like ECS task consolidation) should be avoided given the minimal cost savings involved.

---

## Unresolved Questions & Action Items

1. **Verify recent cost spike correlation**: Confirm whether the large French customer onboarding correlates with the recent CloudWatch cost increase and identify any additional debug logging added around that time. [Owner: Erik Andersson]

2. **Implement logging cleanup**: Remove ~50% of debug logs while preserving critical tracing paths. [Owner: TBD]

3. **Audit caching implementation**: Verify that API credentials, event definitions, and discriminators are cached appropriately with ~10 minute TTLs. [Owner: TBD]

4. **Test Aurora ACU reduction**: Validate that reducing minimum ACUs from 3 to 1-2 does not cause performance degradation. [Owner: TBD]

5. **Separate track - Sync conditions optimization**: Design and implement server-side sync condition evaluation in the CRM system to handle large customers efficiently. [Owner: TBD] [Status: Identified but not in scope for immediate cost reduction]

6. **Customer size approval process**: Establish technical approval gates for large customer onboarding (8M+ contacts) to prevent scaling issues. [Owner: TBD]