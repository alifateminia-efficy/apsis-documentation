---
title: Pipeline Execution Report
run_date: 2026-03-24T10:48:06.023Z
status: SUCCESS
---

# Pipeline Execution Report

## Run Summary

| Parameter | Value |
|-----------|-------|
| **Status** | SUCCESS |
| **Run Date** | 2026-03-24T10:48:06.023Z |
| **Domain** | Apsis One - Integrations |
| **Source Repo** | efficy-sa/apsis-katie |
| **Source Branch** | master |
| **Source Folder** | /apsis-one/kt/integrations |
| **Destination Repo** | alifateminia-efficy/apsis-documentation |
| **Feature Branch** | docs/auto-2026-03-24-1774348828 |
| **Files Processed** | 6 |

## Models Used

| Phase | Model |
|-------|-------|
| Cleanup | claude-haiku-4-5-20251001 |
| Extraction | claude-haiku-4-5-20251001 |
| Synthesis | claude-sonnet-4-6 |
| Self-Review | claude-sonnet-4-6 |

## Token Usage

| Phase | Input Tokens | Output Tokens |
|-------|-------------|---------------|
| Cleanup (total) | 62,566 | 31,688 |
| Extraction (total) | 36,011 | 57,866 |
| Synthesis | 58,139 | 10,078 |
| Self-Review | 11,178 | 2,319 |
| **TOTAL** | **167,894** | **101,951** |

## Per-File Status

| File | Cleanup | Push | Extraction | Details |
|------|---------|------|------------|---------|
| Benjamin - Intergrations Overview.md | OK | OK | OK | - |
| Erik - CRM and pre-fill form.md | OK | OK | OK | - |
| Erik - Justin data recovery.md | OK | OK | OK | - |
| Keyspaces in Integrations part 1.md | OK | OK | OK | - |
| Keyspaces in Integrations part 2.md | OK | OK | OK | - |
| Keyspaces in Integrations part 3 (solutions).md | OK | OK | OK | - |

## Synthesis & Review

| Metric | Value |
|--------|-------|
| Synthesis truncated | No |
| Knowledge base pushed | Yes |
| Review result | pass |
| Critical issues | 1 |
| Modules documented | 20 |
| Business rules captured | 34 |
| Known issues listed | 24 |

## Review Issues

- **[warning]** 3. Module Reference > Connector Libraries: Generic Connector Implementation Guide 2024 update status is flagged as unresolved in both the module reference and Known Issues table, but no actionable resolution path or owner escalation is documented beyond 'Benjamin committed to merging.' If this is a blocker for partner onboarding, the agent needs a decision point, not just a flag.
  - Fix: Add a conditional instruction: 'If onboarding a new partner, verify merge status of 2024 guide with Benjamin or designated owner before proceeding. Do not use 2022 guide for new partners without explicit confirmation.'
- **[critical]** 3. Module Reference > Retry Driver Lambda: Business rule states 'Only opt-out (false) consent messages are redriven; opt-in (true) consent messages are NEVER redriven' but the rationale given ('redrives lose eventual consistency; resetting opt-in status incorrectly is a compliance risk') is partially unclear. Specifically: the mechanism by which opt-in messages are filtered out of the DLQ redrive is not described. An agent cannot implement or audit this without knowing whether filtering happens at the Lambda level, a message attribute check, or DLQ metadata.
  - Fix: Document the filtering mechanism explicitly: e.g., 'Lambda inspects message payload for consent type field X; messages with value true are deleted from DLQ, not requeued.' If unknown, flag as low-confidence with a code-investigation note.
- **[warning]** 4. Cross-Cutting Concerns > Authentication: Internal service-to-service authentication is marked as '⚠️ not fully specified' — this is a significant gap for an AI coding agent that may need to implement or debug inter-service calls. The gap is acknowledged but no investigation path is provided.
  - Fix: Add a directed resolution note: 'Check AWS IAM task roles on ECS task definitions and/or JWT/mTLS configuration in Parameter Store. Escalate to infrastructure owner if unresolved before implementing new inter-service calls.'
- **[warning]** 3. Module Reference > Base Installer: Connector inheritance hierarchy note says 'some connectors skip Generic Connector and inherit directly from Base Installer' without identifying which ones. This is tribal knowledge that could cause an agent to implement a new connector incorrectly by following the wrong inheritance path.
  - Fix: List which connectors inherit directly from Base Installer vs. via Generic Connector, or flag as a required code-archaeology step before implementing any new connector: 'Inspect existing connectors (Dynamics, E-deal, Salesforce, FSC Enterprise, Sideshop, Corporate, Episerver) to determine correct inheritance chain before adding new connector.'
- **[warning]** 6. Known Issues & Workarounds: Two rows in the Known Issues table refer to the Generic Connector Implementation Guide not being merged ('Generic Connector Implementation Guide 2024 updates not merged' and '2022–2024 Generic Connector Implementation Guide not merged'). These appear to be duplicate entries describing the same issue with slightly different wording, creating potential confusion about whether there are one or two distinct problems.
  - Fix: Deduplicate into a single row with a clear description covering both the 2022 baseline and 2024 delta. Clarify whether the 2022 guide is the current operative document or if both need reconciliation.
- **[warning]** 3. Module Reference > Full Sync Consumer: Consumer poll batch size documented as '1 or 10 (configurable ⚠️ — exact default unconfirmed).' For an AI coding agent, an unconfirmed default on a configurable parameter controlling data throughput is a meaningful operational gap, especially if the agent is tasked with tuning or debugging full sync performance.
  - Fix: Flag as requiring code or infrastructure verification: 'Check SQS consumer configuration in Full Sync Consumer ECS task definition or Parameter Store for MaxNumberOfMessages value before making throughput assumptions.'
- **[warning]** 5. Business Rules Reference > Integration Constraints: The rule 'only one CRM integration active per section' references the CRM_ID single field space limitation and a 'Jonas era' origin, but the Known Issues table describes a workaround using bootstrapped unique key spaces (FSC_Enterprise_ID, Dynamics_ID, etc.). It is not explicit whether this workaround fully resolves the one-CRM-per-section constraint or merely defers it. An agent asked to enable a second CRM on a section needs a definitive answer.
  - Fix: Clarify: 'The bootstrapped keyspace workaround allows multiple CRM keyspaces to coexist in Audience, but the CRM_ID single field space means only one CRM integration can write to the legacy CRM_ID field simultaneously. The constraint is not fully lifted; it is structurally deferred. Enabling two simultaneous CRMs requires a migration project not yet scoped.'
- **[warning]** 3. Module Reference > Profile Merge Worker: The message queue type for Profile Merge Worker is flagged as '⚠️ not specified.' The data flow describes an async merge message with a specific schema but the infrastructure substrate is unknown. An agent cannot implement error handling, monitoring, or retry logic for this component without knowing the queue type.
  - Fix: Add a code-investigation directive: 'Determine queue type (SQS standard, SQS FIFO, or Kafka) from Profile Merge Worker service configuration before implementing merge-related error handling or observability.'
- **[suggestion]** 8. Confidence Notes > Low Confidence / Unresolved: 'allow_override_data form option behavior — described as unclear even by Erik; requires code investigation' is listed as low confidence but has no assigned owner, no resolution timeline, and no interim guidance for an agent that encounters this flag in code. This is a form tool interaction that could affect data integrity decisions.
  - Fix: Add interim behavioral assumption for agent use: 'Until resolved, treat allow_override_data as a potential data integrity risk. Do not implement logic that depends on its behavior without first confirming semantics via code inspection of the form submission handler.'
- **[suggestion]** 3. Module Reference > Full Sync Manager: The gotcha 'If the job fails mid-run, infrastructure must be torn down manually before a new job can start' has no runbook reference, no list of affected resources to tear down, and no command or procedure. For an agent handling incident response or full sync failures, this is a critical operational gap.
  - Fix: Add a teardown checklist or reference: 'Manual teardown requires stopping Full Sync Producer and Consumer ECS tasks and clearing the Full Sync Queue (FIFO). Refer to ops runbook [link/location TBD] or contact [owner] for procedure.'
- **[suggestion]** 2. Architecture Map > Data Flow (Forms → CRM): The Forms → CRM data flow references 'Audience Subscription Worker' and 'Form Event Batching Worker' as distinct components, but neither appears in the Components list or has a dedicated Module Reference entry. An agent encountering these names in code or logs has no documentation to reference.
  - Fix: Either add stub Module Reference entries for Audience Subscription Worker and Form Event Batching Worker (even if minimal), or annotate the data flow diagram to clarify these are sub-functions of named modules (e.g., 'Form Event Batching Worker = component of Batch Production Worker for 5-min form window').
- **[suggestion]** 3. Module Reference > Broker Service: The KMS single-region limitation is documented as technical debt, but the mitigation path ('new key + full DB migration') is described without any scope estimate, risk assessment, or decision owner. Given it blocks DR for generic connectors, an agent asked to work on DR scenarios needs to know the current authoritative stance.
  - Fix: Add a DR operational note: 'For DR drills or failover involving generic connectors, current authoritative workaround is customer credential reinstallation. Migration to multi-region KMS key has not been scoped or scheduled. Do not assume this is resolved in DR playbooks.'
- **[suggestion]** 4. Cross-Cutting Concerns > Error Handling: Form submission 504 errors from CRM are noted with a retry logic flag but the backoff strategy is unspecified. Given form submissions are part of the Forms → CRM data flow and feed Profile Merge Worker, an unspecified retry strategy is a latent correctness risk.
  - Fix: Flag for code investigation: 'Inspect Form Event Batching Worker or Outbound Worker retry configuration for CRM 504 handling. Document actual backoff parameters before implementing any changes to form submission error handling.'