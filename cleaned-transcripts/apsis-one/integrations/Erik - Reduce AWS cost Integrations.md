---
source_file: Erik - Reduce AWS cost Integrations.txt
domain: Apsis One Integrations
topics: [AWS cost reduction, CloudWatch logging optimization, Aurora Serverless RDS scaling, sync conditions architecture, large customer data processing, ECS microservice consolidation]
speakers: [Grzegorz Kozub (Speaker 1 / likely Grzegorz), Erik Andersson, Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [CloudWatch, Aurora Serverless (RDS), ECS tasks, CRM sync pipeline, sync conditions engine, consent/profile update flows]
session_type: knowledge-transfer
---

## Session Overview

This session is a focused cost-reduction review for the Apsis One Integrations domain on AWS. The monthly cost for the Integrations domain is approximately **$1,300 USD**, with the two largest cost drivers identified as **RDS (Aurora Serverless)** and **CloudWatch**. Erik Andersson, as the domain expert, provided concrete low-hanging-fruit recommendations for both. The session also surfaced a significant architectural issue — large customers triggering full syncs of millions of contacts due to sync conditions being evaluated inside Apsis rather than at the CRM — which is both a cost driver and a scalability blocker. The session concluded with a brief discussion on ECS microservice consolidation, which was ruled out as not worth the development investment.

---

## Context and Cost Proportionality

- Total monthly integration cost: **~$1,300 USD**
- This domain is approximately 1/120th of the broader (audience) cost
- The domain is considered **already reasonably optimized** at this point
- Focus is on low-hanging fruit — quick wins that don't require deep architectural changes
- The two most expensive AWS services in this domain: **RDS** and **CloudWatch**

---

## CloudWatch Log Reduction

### What Must Be Retained

[Erik Andersson]: Certain logs are **non-negotiable** for support and traceability:
- CRM contact IDs must be traceable end-to-end — a very frequent support question is "why didn't this CRM contact get synced / get consent?"
- Required log events include:
  - "We received a message for this contact"
  - "We successfully processed this contact for consent / profile update"
  - "This profile is now entering this flow"
  - "This profile exited this flow"

### What Can Be Removed

[Erik Andersson]: There is a large volume of debug/info logging that has accumulated because, historically, the cost to clean it up exceeded the savings. Now that cost reduction is a goal, these are worth removing:

- Logging of **configuration state** (e.g., what mappings are currently active)
- **Debug logs for requests**
- Various other `INFO`-level logs that were added during development

> "I would almost go so far as to say you can probably remove half of the logs we have already."

This is described as a **very low-hanging fruit**.

---

## Aurora Serverless RDS — Reducing Minimum ACUs

### Current Configuration

- Using **Aurora Serverless**
- Currently configured with **3 ACUs (Aurora Capacity Units) as the minimum**

### Recommendation

[Erik Andersson]: Reduce the minimum ACUs to **2 or even 1**, because:

- Integrations is not inherently a database-heavy domain
- Most repeated data lookups **should be cached in memory**, examples:
  - API credentials per customer — fetched once, used 40,000 times
  - Event definition IDs / discriminators — fetched once, used 400,000 times
- The expected access pattern is: fetch at the start of a full sync or when a real-time message arrives, then hold in memory — "once every ~10 minutes" is the rough expectation

> "If you notice that there isn't [a cache for something], then there should be."

### Caveat

[Erik Andersson] acknowledged this is somewhat ironic given the caching philosophy, but the principle is sound: database load should be minimal if caching is implemented correctly. If ACU reduction causes issues, it would be a signal that caching has gaps that should be fixed anyway.

---

## Large Customer Sync — Root Cause of CloudWatch Spike

### The Problem

A large French customer (and potentially others of similar scale) is causing a significant overhead in CloudWatch logs, CPU usage, and sync duration.

- Syncs are currently running for approximately **1.5 days** with high CPU
- The cost spike in CloudWatch is suspected to correlate with this customer's onboarding

### Architectural Root Cause: Sync Conditions Evaluated Inside Apsis

[Erik Andersson]: The core architectural issue is that **sync conditions are evaluated inside Apsis**, not at the CRM. This means:

1. A customer can configure a filter, e.g., "only sync contacts based in Sweden"
2. The CRM has no awareness of this filter
3. Apsis must **download the entire contact database** from the CRM to apply the filter
4. Concrete example:
   - Customer has **8 million contacts** total in their CRM
   - Only **100,000 are based in Sweden** (the target segment)
   - Apsis downloads all 8 million, processes them, and discards ~98%
   - All 8 million contacts generate log entries, even the discarded ones

> "This is an insane way of doing it. We haven't technically approved these sizes of customers, but it has just happened."

- Similar problem exists for consents: processing **16 million consents**, discarding ~98%

### Impact

- Massive CloudWatch log volume (multiplied by contact count, including discarded contacts)
- Sync memory pressure — syncs die due to memory exhaustion
- CRM systems **time out after 1.5 days** because they are being overloaded

### Planned Fix (Separate Track)

[Erik Andersson]: The fix is to **push sync conditions down to the CRM** so that:
- Instead of downloading 8 million contacts, Apsis is only presented with the 100,000 that match
- This is described as an optimization "by a factor of an insane amount"

This is a **necessary track** to handle large customers at all, but is explicitly called out as **separate from the CloudWatch cleanup work**. Both tracks help costs, but they are independent efforts.

### Short-Term Mitigation

Removing the excess debug/info logs (described above) will reduce CloudWatch costs from these large syncs in the interim, since log volume is directly multiplied by contact count.

[Erik Andersson] also noted there may have been an **unfortunate debug log added in a recent release** around the time the spike appeared — this needs to be investigated by checking what was released at that time.

---

## ECS Microservice Consolidation — Ruled Out

[Erik Andersson]: Currently, every service in the Integrations domain runs as a **separate ECS task** (microservice architecture). The theoretical option of consolidating them into a single ECS task was considered.

**Conclusion: Not worth doing.**

Reasons:
- High development cost
- Would make the system more complicated
- Cost savings would not be substantial enough to justify the investment

> "I think you will lose money in that aspect... It will make it more complicated, in all honesty."

---

## Key Takeaways

1. **CloudWatch is the primary optimization target.** Roughly half of current log statements can be removed — focus on debug/info logs for configuration state and request details, while preserving traceability logs for CRM contact flows.
2. **Aurora Serverless minimum ACUs can likely be reduced from 3 to 2 or 1**, given that caching should be absorbing the vast majority of repeated DB reads. If it causes issues, that reveals a caching gap that should be fixed.
3. **Large customer full syncs are a major cost multiplier.** The sync conditions architecture (filtering inside Apsis rather than at the CRM) causes Apsis to process and log millions of contacts that are ultimately discarded. This is both a cost issue and a scalability blocker.
4. **Two separate tracks** address large customer costs: (a) short-term log cleanup, and (b) longer-term architectural change to push sync conditions to the CRM side.
5. **ECS consolidation is not recommended** — the development cost exceeds the savings and adds complexity.
6. Other AWS costs beyond RDS and CloudWatch are not significant enough to warrant investigation.

---

## Unresolved Questions / Action Items

- [ ] **Identify and remove excessive debug/info logs** in the Integrations codebase — estimated ~50% of current log volume is removable. Be careful to preserve CRM ID traceability and flow entry/exit logs.
- [ ] **Investigate recent release** around the time of the CloudWatch cost spike — a debug log may have been accidentally left in that is amplifying costs for large customer syncs.
- [ ] **Evaluate reducing Aurora Serverless minimum ACUs** from 3 → 2 or 1; validate that in-memory caching for API credentials and event definition IDs is functioning correctly before reducing.
- [ ] **Architectural work: push sync conditions to CRM systems** — this is a separate, longer-term track but is required to make large customers viable at all (memory exhaustion and CRM timeouts make the current approach unsustainable).
- [ ] ⚠️ **[AMBIGUOUS]** The "French customer" is referenced as a specific known case, but it is unclear whether there are other customers of similar scale already onboarded that are contributing to the same pattern. The scope of the large-customer problem should be assessed.