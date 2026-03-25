---
source_file: Erik - Reduce AWS cost Integrations.txt
domain: Apsis One Integrations
topics: [AWS Cost Optimization, CloudWatch Logging, RDS Database Optimization, Sync Conditions Performance, Caching Strategy]
speakers: [Speaker 1 (likely Product/Engineering Lead), Erik Andersson (Integration Domain Expert), Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [CloudWatch, RDS Aurora Serverless, ECS Tasks, Sync Conditions, CRM Integration, API Credentials Caching]
session_type: cost-optimization-review
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This session focused on identifying low-hanging fruit for reducing AWS costs in the Apsis One Integrations domain, which currently costs approximately $1,300 USD monthly. The integration domain represents only 1/120th of total platform costs, but optimization opportunities were identified in two primary areas: CloudWatch logging overhead and RDS Aurora Serverless configuration. The discussion highlighted how sync conditions filtering strategy and caching implementation significantly impact infrastructure costs, particularly for large customers.

---

## Current Cost Baseline and Priority Areas

The integration domain operates at approximately **$1,300 USD monthly** in AWS costs. This represents a relatively small portion of the platform's overall cloud spend, which informed the approach to focus on quick wins rather than deep optimization work.

[Speaker 1]: The most expensive services identified through cost explorer are **RDS** and **CloudWatch**, with CloudWatch being particularly significant due to excessive logging volume.

The team acknowledged that the integration domain is already reasonably optimized in its current state, so the focus was intentionally narrow.

---

## CloudWatch Logging Optimization

### Current Logging Strategy and Necessity

[Erik Andersson]: There is a critical distinction between logs that must be retained and debug logs that can be removed. Certain logging is mandatory for operational tracing:

> We always use like CRM IDs that come in from the system and a very frequent question is why didn't this profile that is like this contact from the CRM? Why didn't that get synced to apps? Why didn't it get consent. For that we need to be able to trace the CRM IDs.

The minimal required logging includes:
- CRM ID tracing (for support and debugging customer sync issues)
- Message receipt confirmation logs
- Contact processing status (successful/failed processing for consent and profile updates)
- Flow entry and exit events

### Logs That Can Be Removed

[Erik Andersson]: Beyond the mandatory tracing logs, there is substantial debug logging that accumulated over time because optimization costs were not justified:

> There is a lot of debug things in the logs which have been there because as you said, like looking at the cost for integration, it has almost been like if it will cost us more time to spend to optimize the logs than we would save it.

Specific candidates for removal:
- Configuration status logs
- Mapping information logs
- General debug logs for requests
- Info-level logs that provide non-critical operational details

**Estimated impact**: [Erik Andersson] suggests approximately **50% of current logs could be removed** without losing critical operational visibility.

### Interaction with Sync Conditions and Large Customer Impact

The logging volume is multiplicatively affected by the sync conditions implementation issue. When a customer has 8 million total contacts but only wants to sync 100,000 (filtered by geography, status, etc.), the current implementation:
- Downloads all 8 million contacts from the CRM
- Processes all 8 million through sync condition evaluation
- Logs entries for all 8 million contacts
- Discards 98% of them after logging

[Erik Andersson]: This creates a vicious cycle where logging costs scale with total CRM dataset size, not actual synced record count:

> The logs here will be multiplied of course by the amount of contacts in the sync, even if we at a later stage discard them. So for these big customers, we are processing 8 million contacts of which we sink 100,000 and this is an insane way of doing it.

---

## RDS Aurora Serverless Optimization

### Current Configuration

The integration domain currently uses **Aurora Serverless** with a minimum of **3 ACUs (Aurora Capacity Units)** provisioned.

[Erik Andersson]: The integration domain should not be heavily database-dependent and can likely operate with reduced capacity:

> Integrations should in theory not be a very database heavy domain. This is going to be very ironic coming from me now, but we should have caches for most things.

### Recommended Configuration Changes

**Reduce minimum ACU count from 3 to 1 or 2**. The reasoning:
- Aurora Serverless auto-scaling handles variable load
- Integration workloads are not primarily database-intensive
- Caching strategy (see below) reduces database query frequency

This change was deemed lower-risk than other optimizations because the database will automatically scale up during demand spikes.

---

## Caching Strategy for API Credentials and Metadata

### Problem Statement

The integrations domain makes redundant database and external API calls for data that changes infrequently.

[Erik Andersson]: The same data is retrieved thousands or hundreds of thousands of times:

> We need to retrieve like API credentials for a customer and then use that like 40,000 times. We need to get events definition ID's or discriminators or whatever and then use that 400,000 times.

### Required Caching Implementation

**API Credentials**: Retrieved once per sync or per time period, should remain in memory for all subsequent operations within that sync cycle.

**Event Definitions and Discriminators**: Should be cached in-memory and refreshed only periodically (suggested: every 10 minutes), not on every request or every contact.

[Erik Andersson]: The cache invalidation strategy should balance freshness with cost:

> We should only need to get it like once every say 10 minutes or something. It should remain in memory when it is fetched, so you should not need a very database or not need a lot of database green.

**Where caching is needed**: Start of full sync operations and at the beginning of real-time message processing batches.

**Current gap**: This caching implementation is not currently in place for all necessary data types, and identifying and implementing these caches is necessary to reduce database load.

---

## Sync Conditions Filter Push-Down to CRM

### The Problem with Current Implementation

Sync conditions (e.g., "only sync Swedish customers," "only active contacts") are evaluated **inside Apsis** after data retrieval, not at the CRM source system.

[Erik Andersson]: The flow currently works as follows:
1. CRM has 8 million total contacts
2. Apsis requests all 8 million contacts
3. Apsis evaluates sync conditions (e.g., country = Sweden)
4. Apsis discards 7.9 million contacts that don't match conditions
5. Apsis syncs 100,000 contacts that match

This is inefficient across multiple dimensions:
- **Network bandwidth**: Unnecessary data transfer
- **CPU time**: Processing millions of contacts only to discard them
- **Memory**: Loading 8 million contact records
- **Logging overhead**: CloudWatch charges multiply by full dataset size, not filtered result set
- **CRM timeout risk**: Large requests cause the source CRM to become overloaded for extended periods (e.g., 1.5 days)

### Proposed Solution

Push sync conditions down to the CRM system so that Apsis only receives the filtered result set (100,000 contacts instead of 8 million).

[Erik Andersson]: This is a separate optimization track from logging cleanup, but is critical for large customers:

> It's a necessary track, but that's for us to be able to handle these big customers because we can't, we can't do it otherwise like the sinks die because of memory and the CRMS time out for 1 1/2 day because they are getting overloaded.

**Implementation status**: This optimization is under active consideration but has not yet been implemented. It represents a significant development effort.

**Impact on current cost issues**: This may partially explain the cost spike for certain large customers, though additional debug logging introduced in a recent release may also be contributing.

---

## Large Customer Case Study

A significant customer (referenced as "the French customer") is driving disproportionate costs due to the sync conditions problem.

**Characteristics**:
- Large contact dataset (8 million contacts or more)
- Sync conditions filtering to small subset (e.g., Swedish region only = 100,000 contacts)
- Full sync operations running for 1.5 days with high CPU utilization
- Creates "a lot of overhead in terms of everything, basically mainly traffic and that then converts to the log messages"

[Erik Andersson]: The cost impact is reflected in CloudWatch logs and CPU usage, though the exact correlation has not been precisely measured:

> It is an issue in the sense that it will drive up the costs for the sync. Like what is happening right now is that the sync are running for like 1 1/2 day with quite a lot of CPU power that I don't think that's reflected in your logs, but for sure this is going to affect.

**Note**: A separate customer issue emerged during this session that required immediate attention, preventing full analysis of this case.

---

## Microservice Architecture Cost Considerations

### Current Design

Every service in the integration domain runs as a separate **ECS task** operating as a microservice.

### Consolidation Consideration

[Erik Andersson]: Combining integration services into a single ECS task was considered but rejected:

> If you would want to really fine tune it, I guess you can like combine integration to 1 task, but this is gonna be a lot of development time, so I think you will lose money in that aspect.

[Speaker 1]: The cost gains from microservice consolidation would not be substantial:

> The gains from the other not substantial at all.

**Conclusion**: Maintaining the current microservice architecture is preferable to the development cost of consolidation. Other optimizations offer better ROI.

---

## Recommended Action Items and Priority

### High Priority (Low Effort, Clear ROI)
1. **Remove non-essential debug logs from CloudWatch** - Remove ~50% of current logging while retaining mandatory CRM ID tracing and operational events
   - Effort: Low
   - Expected impact: Significant CloudWatch cost reduction
   - Implementation owner: Erik Andersson to identify specific log statements

2. **Reduce Aurora Serverless minimum ACU from 3 to 1 or 2**
   - Effort: Very Low (configuration change)
   - Expected impact: Moderate RDS cost reduction
   - Risk: Low (auto-scaling handles spikes)

3. **Implement in-memory caching for API credentials and event definitions**
   - Effort: Medium
   - Expected impact: RDS query reduction, faster sync performance
   - Implementation owner: Integration team

### Medium Priority (High Effort, Critical for Scale)
4. **Push sync conditions filtering to CRM source system**
   - Effort: High (architectural change)
   - Expected impact: Transformational for large customers (10x+ improvement in sync efficiency)
   - Current blocker: Large customer scale not technically approved; current implementation breaks at large volumes
   - This is necessary for handling customers like the 8M-contact case

### Not Recommended
- Microservice consolidation (development cost exceeds savings)

---

## Key Takeaways

1. **Two primary cost drivers identified**: CloudWatch logging and RDS capacity, representing ~80% of integration domain costs.

2. **Logging optimization is the clearest quick win**: Approximately 50% of CloudWatch logs are debug-level and can be safely removed while retaining all operationally necessary CRM tracing.

3. **Sync conditions filtering is a critical architectural gap**: Processing 8M contacts to sync 100K is unsustainable for large customers and directly drives CloudWatch costs. This must be pushed down to the CRM source system.

4. **Caching is underutilized**: API credentials and metadata definitions that are accessed hundreds of thousands of times per sync should be cached in-memory rather than queried repeatedly from the database.

5. **Current architecture is reasonable**: The microservice approach and existing service design do not need to be refactored; targeted optimizations in logging, caching, and filtering strategy are sufficient.

6. **Cost/benefit is favorable for identified changes**: The recommended optimizations have low implementation cost relative to savings and improve platform performance as a side benefit.

---

## Unresolved Questions and Gaps

1. **Exact correlation of large customer onboarding to cost spike**: The timing of when the French customer began processing relative to the CloudWatch cost increase has not been precisely determined. This would help quantify the impact of sync conditions filtering vs. recent debug logging changes.

2. **Specific debug log statements to remove**: Erik indicated which categories of logs can be removed but did not provide the exact code locations. This requires detailed code review.

3. **Customer scale approval limits**: Integration domain has not officially approved customers of the 8M-contact size, yet they are operational. There may be other undocumented large customers.

4. **CRM filter push-down timeline**: No timeline or resource allocation was discussed for the sync conditions optimization, which is blocking customer scale.