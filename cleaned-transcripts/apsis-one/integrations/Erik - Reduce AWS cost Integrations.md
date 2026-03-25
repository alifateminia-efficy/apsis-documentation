---
source_file: Erik - Reduce AWS cost Integrations.txt
domain: Apsis One Integrations
topics: [AWS Cost Optimization, CloudWatch Logging, RDS Database Optimization, Sync Conditions, Large Customer Data Processing, Caching Strategy]
speakers: [Speaker 1 (likely Grzegorz Kozub), Erik Andersson, Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [CloudWatch, RDS/Aurora Serverless, ECS Tasks, CRM Integration, Sync Conditions, Caching Layer, API Credentials]
session_type: cost-optimization-review
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This is a focused AWS cost optimization session for the Apsis One Integrations domain. The team identifies that monthly integration costs are approximately $1.3K USD, which represents a small fraction of total platform costs. The discussion centers on two primary cost drivers: CloudWatch logging and RDS database usage. Key opportunities identified include reducing unnecessary debug logging, optimizing Aurora Serverless instance counts, implementing caching for frequently-accessed data, and addressing a specific large customer causing exceptional processing overhead due to inefficient sync condition evaluation.

---

## Current Cost Landscape

**Monthly integration costs:** $1.3K USD (representing approximately 1/120th of total audience costs, which is the larger cost driver)

The team notes that the integration domain is already fairly well optimized from a cost perspective, so the focus is on identifying low-hanging fruit rather than major architectural changes.

**Primary cost drivers identified:**
- **CloudWatch Logs** — identified as the most expensive service
- **RDS** — second most expensive service

Other AWS services do not show significant cost outliers and are not prioritized for optimization.

---

## CloudWatch Logging Optimization

### Current Logging Requirements

Erik Andersson emphasizes that certain logging is non-negotiable for operational visibility:

> "There are some things which must remain for tracing. We always use like CRM IDs that come in from the system and a very frequent question is why didn't this profile that is like this contact from the CRM? Why didn't that get synced to apps? Why didn't it get consent. For that we need to be able to trace the CRM IDs."

**Essential logging that must be preserved:**
- CRM IDs for traceability
- Contact sync status messages
- Successful processing confirmations
- Profile flow entry/exit events
- Consent processing records

### Logging Optimization Opportunity

Despite these core requirements, there is substantial opportunity to reduce logging volume:

> "There is a lot of other things we don't need. There is a lot of debug things in the logs which have been there because... if it will cost us more time to spend to optimize the logs than we would save it, but... you can remove a lot of the info logs we have. We are logging like the status of like some configuration like what mappings and some debug other debug logs for the request that we have that you can get rid of. I would almost go so far as to say you can probably remove half of the logs we have already."

**Specific debug logging candidates for removal:**
- Configuration status logs
- Mapping details logs
- Request-level debug logs
- Info-level logs that don't contribute to traceability

Erik estimates that approximately **50% of current logging could be removed** without impacting operational observability or debugging capability.

### Logging Impact on Large Customer Processing

The logging volume issue is exacerbated when processing large customer datasets. See section below on "Large Customer Processing and Sync Conditions" for how logging multiplies with volume.

---

## RDS and Database Optimization

### Aurora Serverless Configuration

The integration domain currently uses **Aurora Serverless with 3 ACUs (Aurora Capacity Units) as minimum**.

Erik suggests this can be optimized:

> "You can look into reducing that down to two or even one maybe because it, I mean it is scaling, but integrations should in theory not be a very database heavy domain."

**Recommendation:** Evaluate reducing minimum ACU count from 3 to 2, or potentially to 1, as the integrations domain should not be database-heavy under normal conditions.

### Caching Strategy for Reduced Database Load

Erik identifies a critical pattern where the domain fetches the same data repeatedly:

> "This is going to be very ironic coming from me now, but we should have caches for most things. And if you notice that there isn't, then there should be because like we need to retrieve like API credentials for a customer and then use that like 40,000 times. We need to get events definition ID's or discriminators or whatever and then use that 400,000 times."

**Specific caching opportunities:**
- **API credentials per customer:** Currently fetched multiple times when could be cached in memory; refresh rate should be ~10 minutes
- **Event definition IDs:** Retrieved repeatedly within a sync cycle; should be cached once per sync or on a time-based refresh
- **Discriminators:** Similar pattern to event definition IDs

**Caching usage patterns:**
- Initially fetched at the start of a full sync
- Retrieved again if real-time messages arrive during sync
- Should persist in memory between retrievals
- Suggests 10-minute TTL as reasonable cache expiration strategy

This caching approach would significantly reduce database queries without impacting functionality.

---

## Large Customer Processing and Sync Conditions

### The Sync Conditions Problem

The team has identified a specific customer case (referenced as "French customer") that creates disproportionate operational overhead. The root cause is in how **sync conditions** are currently evaluated:

> "Because the sync conditions are evaluated inside of Apsis. The CRM doesn't know about this. So let's say now that they have one section where they only want to have a Swedish customer and they have 8 million contacts, but they only have 100,000 that are based in Sweden. I still need to download all 8 million and process them."

**Current inefficient workflow:**
1. Integration system downloads all 8 million contacts from CRM
2. Integration system filters/processes all 8 million contacts locally
3. Integration system applies sync conditions (e.g., "country == Sweden")
4. Only ~100,000 contacts (1.25%) are ultimately synced to Apsis
5. All 8 million are logged, creating massive CloudWatch overhead

**Impact of this pattern:**
- Extreme CPU waste processing contacts that will be discarded
- Massive logging volume (multiplied by 8 million contacts)
- Extended processing time (sync runs for ~1.5 days with high CPU usage)
- CRM system becomes overloaded, timing out from long-running queries
- This is particularly severe during full sync operations

### Example Scale Issues

The team references scenarios with even larger datasets:
- 16 million consents where 98% are ultimately discarded
- Multiple sync sections (e.g., separate sections for Sweden, Denmark, Finland)

> "For these big customers, we are processing 8 million contacts of which we sink 100,000 and this is an insane way of doing it. Like we haven't technically approved these sizes customers, but it has just happened now."

### Planned Solution: Push Filtering to CRM

The team is investigating moving sync condition evaluation to the CRM system side:

> "So the sync conditions will be sent to the CRM system. Instead of us downloading 8 million contacts, we will only be presented with the 100,000 that will be included there."

**Expected benefits:**
- CRM applies filtering before data transfer (reduces network overhead)
- Integration system receives only relevant contacts (~100,000 instead of 8 million)
- Logging volume reduced by factor of 80x (proportional to contacts processed)
- CPU usage dramatically reduced
- Sync duration reduced from ~1.5 days to reasonable timeframe
- CRM system no longer overloaded

**Status:** This is identified as a "necessary track" for handling large customers but is separate from immediate logging cleanup efforts. The feature is required because "we can't do it otherwise like the sinks die because of memory and the CRMs time out for 1 1/2 day because they are getting overloaded."

### Interaction with Debug Logging Issue

It's suspected that a recent code release added debug logging that compounds this problem:

> "I don't think this is the whole reason for the spike. We must have added some more debug log that is unfortunate, which should be removed and I need to check in what we released on around that time."

**Action item:** Erik to review recent releases to identify and remove any debug logging additions that coincide with the cost spike.

---

## ECS Task Architecture Considerations

### Current Microservice Structure

Erik notes the current architecture approach:

> "We have every service in integration right now is essentially a microservice with in an ECS task."

### Potential Consolidation and Cost Impact

When asked about combining multiple services into a single ECS task to reduce costs, Erik provides this assessment:

> "If you would want to really fine tune it, I guess you can like combine integration to 1 Task, but this is gonna be a lot of development time, so I think you will lose money in that aspect."

**Conclusion:** Consolidating ECS tasks would provide negligible cost savings while introducing significant development overhead and increased complexity. This optimization is **not recommended** for pursuit at this time.

---

## Key Takeaways

1. **CloudWatch logging cleanup is the quickest win:** Approximately 50% of current logs can be removed without impacting traceability. Focus on eliminating configuration status, mapping details, and request-level debug logs while preserving CRM ID tracing, sync status, and consent processing logs.

2. **Caching is underutilized:** Implement in-memory caching for API credentials, event definition IDs, and discriminators with appropriate TTLs (~10 minutes). This will substantially reduce RDS load without architectural changes.

3. **Aurora Serverless scaling down is safe:** Reduce minimum ACU count from 3 to 2 or 1, as the integrations domain should not be database-heavy under normal caching conditions.

4. **Large customer sync conditions are a separate optimization track:** Moving sync condition evaluation to the CRM system is necessary to handle existing large customers (8M+ contacts) but requires feature development. This will dramatically reduce CPU, logging, and duration issues. Do not block immediate logging cleanup on completion of this feature.

5. **Logging volume scales with contact count:** For large customers, logging cleanup becomes increasingly impactful as the volume of processed contacts grows.

6. **ECS task consolidation is not cost-effective:** The development effort required to merge microservices into single ECS tasks will exceed cost savings.

---

## Unresolved Questions & Action Items

- **Erik to review recent code releases** around the time of the cost spike to identify and remove any debug logging additions that correlate with increased CloudWatch costs
- **Specific behavior of the "French customer" case:** While the large-customer sync condition pattern is understood, the transcript notes that this customer experiences recurring issues even after fixes, suggesting additional investigation may be needed
- **Exact caching implementation details:** The session identifies where caching should exist but doesn't specify current implementation status or TTL policies; audit needed to confirm caches are actually in place