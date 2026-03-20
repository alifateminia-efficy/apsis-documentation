---
title: Pipeline Execution Report
run_date: 2026-03-20T09:54:01.337Z
status: COMPLETED_WITH_WARNINGS
---

# Pipeline Execution Report

## Run Summary

| Parameter | Value |
|-----------|-------|
| **Status** | COMPLETED_WITH_WARNINGS |
| **Run Date** | 2026-03-20T09:54:01.337Z |
| **Domain** | Apsis One - Integrations |
| **Source Repo** | efficy-sa/apsis-katie |
| **Source Branch** | master |
| **Source Folder** | /apsis-one/kt/integrations |
| **Destination Repo** | alifateminia-efficy/apsis-documentation |
| **Feature Branch** | docs/auto-2026-03-20-1773999810 |
| **Files Processed** | 38 |

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
| Cleanup (total) | 378,949 | 178,973 |
| Extraction (total) | 206,317 | 370,979 |
| Synthesis | 380,120 | 12,049 |
| Self-Review | 13,182 | 3,000 |
| **TOTAL** | **978,568** | **565,001** |

## Per-File Status

| File | Cleanup | Push | Extraction |
|------|---------|------|------------|
| Benjamin - Intergrations Overview.md | OK | OK | PARSE_ERROR |
| Discuss display names for connectors PR changes.md | OK | OK | OK |
| Efficy Enterprise issues.md | OK | OK | OK |
| Erik - 20.11.2025.md | OK | OK | OK |
| Erik - Accessing customer CRM for manual debug and incident recovery.md | OK | OK | OK |
| Erik - CRM and pre-fill form.md | OK | OK | PARSE_ERROR |
| Erik - Connector API & CICD.md | OK | OK | OK |
| Erik - Dynamics setup part 2 & code review of event listeners for campaigns.md | OK | OK | OK |
| Erik - Full sync issue.md | OK | OK | OK |
| Erik - Handling Sync Conditions in Efficy Enterprise.md | OK | OK | OK |
| Erik - Inbound Flow.md | OK | OK | OK |
| Erik - Incidents in Integrations.md | OK | OK | OK |
| Erik - Integration statistics for Back Office kick-off.md | OK | OK | OK |
| Erik - Investigate and design solution for approximate age of oldest message.md | OK | OK | OK |
| Erik - Justin data recovery.md | OK | OK | PARSE_ERROR |
| Erik - Leads.md | OK | OK | OK |
| Erik - Local development.md | OK | OK | OK |
| Erik - Microsoft Dynamics.md | OK | OK | OK |
| Erik - Outbound flow.md | OK | OK | OK |
| Erik - Plugins and Tribe lead creation issue.md | OK | OK | OK |
| Erik - Reduce AWS cost Integrations.md | OK | OK | OK |
| Erik - Refactor Justin connectivity flows.md | OK | OK | OK |
| Erik - Squid proxy.md | OK | OK | OK |
| Erik - Sync conditions filtering.md | OK | OK | OK |
| Erik - Types of Connectors.md | OK | OK | OK |
| Erik from Dec 11th 2025 (continued).md | OK | OK | OK |
| Erik from Dec 11th 2025.md | OK | OK | OK |
| Erik from Nov 27th 2025.md | OK | OK | OK |
| Findings of Enterprise issue.md | OK | OK | OK |
| Integration installation story grooming.md | OK | OK | OK |
| Integration projects never finished due to FE resources.md | OK | OK | OK |
| Keyspaces in Integrations part 1.md | OK | OK | OK |
| Keyspaces in Integrations part 2.md | OK | OK | PARSE_ERROR |
| Keyspaces in Integrations part 3 (solutions).md | OK | OK | PARSE_ERROR |
| New feature - Duplex integration.md | OK | OK | OK |
| Shreevidhya - Efficy Enterprise 12.1.md | OK | OK | OK |
| Testing & deploying Justin part 1.md | OK | OK | OK |
| Testing & deploying Justin part 2.md | OK | OK | OK |

## Synthesis & Review

| Metric | Value |
|--------|-------|
| Synthesis truncated | No |
| Knowledge base pushed | Yes |
| Review result | pass |
| Critical issues | 1 |
| Modules documented | 23 |
| Business rules captured | 31 |
| Known issues listed | 21 |

## ⚠️ Extraction Parse Errors

- Benjamin - Intergrations Overview.md
- Erik - CRM and pre-fill form.md
- Erik - Justin data recovery.md
- Keyspaces in Integrations part 2.md
- Keyspaces in Integrations part 3 (solutions).md

## Review Issues

- **[warning]** Generic Connector - Key files: URL 'https://integration-files.appsis.one/' contains a probable typo ('appsis' instead of 'apsis'). This could mislead an agent trying to access the spec.
  - Fix: Verify the correct URL. Likely should be 'https://integration-files.apsis.one/' — correct or flag with a ⚠️ confidence note.
- **[warning]** 3. Module Reference - Delta Sync Worker (DSW): The DSW section states outbound calls go 'through Squid proxy' but the architecture map shows DSW calls Apsis HLS/Consent APIs (internal), not external CRM. The Squid proxy is for outbound CRM traffic only. This is contradictory.
  - Fix: Clarify which DSW calls go through Squid (FCC 12.0 full contact fetch callback does; HLS/Consent API calls do not). Remove or qualify the parenthetical '(outbound through Squid proxy)' under Integration points.
- **[warning]** 4. Cross-Cutting Concerns - Error Handling: SQS exponential backoff sequence listed as '~2s, 1m, 5m, 15m, 1h, 3h, 5-6h' but the Outbound Worker module documents it as '15s, 15s, 1m, 5m, 15m, 1h, 4h, 6h'. These two sequences are inconsistent within the same document.
  - Fix: Reconcile the two backoff sequences. The Outbound Worker section is more detailed and likely more accurate. Update the Cross-Cutting Concerns section to match, or flag the discrepancy explicitly in Section 8 Conflicts.
- **[warning]** 3. Module Reference - Broker Service - Configuration: Broker internal URL is listed as 'broker-internal.apsis.io' but the document elsewhere uses 'apsis.cloud' and 'apsis.one' as domain patterns. The '.io' TLD is inconsistent and may be incorrect.
  - Fix: Verify the actual internal broker service URL against Squid allowlist config or service discovery records and correct if needed. Flag with ⚠️ if unverifiable.
- **[warning]** 3. Module Reference - Connector Libraries (CRM-specific) - Gotchas: Text references 'FSC Enterprise 12.0 returns integers as stringified '123'' but uses curly/smart quotes rather than straight quotes. For an AI coding agent this could cause copy-paste or parsing issues in code contexts.
  - Fix: Replace all curly/smart quotes in code-referencing contexts with straight ASCII quotes.
- **[warning]** 3. Module Reference - Full Sync Manager (FSM) - Business rules: The note '⚠️ Benjamin acknowledges "a bit overkill"' attributes an opinion to a named individual. This is informal tribal knowledge that could become stale or misleading (especially noted in Section 8 that Erik Andersson is leaving — personnel churn is a risk).
  - Fix: Replace named attribution with a neutral architectural note, e.g., 'Acknowledged as potentially over-engineered; dedicated stack per job was chosen to avoid legacy FQSOA contention issues.'
- **[suggestion]** 3. Module Reference - Sluice Worker (SLW) - Gotchas: The quote 'a solution we wish we didn't have' is informal and does not add actionable value for an AI agent. It may cause the agent to under-value this component.
  - Fix: Replace with a neutral architectural note explaining why the pattern exists and what the preferred future state is, e.g., 'Acknowledged technical debt; preferred solution is atomic full sync with real-time pause at CRM webhook level.'
- **[critical]** 3. Module Reference - Full Sync Producer - Gotchas: 'Do NOT cache contacts in-memory for comparison (crashes at 2M+ contacts). Must stream messages to SQS immediately. In-memory cache removal is pending implementation.' — This states a critical constraint as both a rule AND an unimplemented fix simultaneously. If the cache has NOT been removed yet, the code still has the bug. An agent modifying the producer could be misled about the current code state.
  - Fix: Clearly separate current state from desired state: 'CURRENT STATE: in-memory cache exists and causes crashes at 2M+ contacts (known bug, fix in progress). DO NOT add further in-memory accumulation. TARGET STATE: stream directly to SQS without caching.' Add to Known Issues table if not already tracked.
- **[warning]** 6. Known Issues & Workarounds: Issue 'In-memory audience export comparison cache crashes for 8M+ contacts' lists threshold as 8M+, but the Full Sync Producer Gotchas section states 2M+. There is a numeric discrepancy between these two descriptions of what appears to be the same or related issue.
  - Fix: Clarify whether the 2M+ threshold applies to Full Sync Producer sync conditions crash and the 8M+ threshold applies specifically to the audience export cache, or whether one is incorrect. Add explicit disambiguation.
- **[warning]** 3. Module Reference - Audience Subscription Worker - Gotchas: Example given: 'Excel converts large integers to floats: 123,500,000 becomes invalid'. The number format uses a comma as thousands separator which could be misread as a decimal separator in some locales/contexts. This is ambiguous for an agent.
  - Fix: Use unambiguous notation: '123500000 becomes 1.235E+8 (float)' or similar.
- **[warning]** 3. Module Reference - Disaster Recovery (DR) / Deployment - Gotchas: 'System hasn't been deployed from scratch in 2+ years—expect undocumented dependencies.' This is a critical operational risk with no actionable mitigation documented beyond the vague 'expect undocumented dependencies'.
  - Fix: Add a recommended pre-DR checklist item: 'Run a DR dry-run in staging annually and document any new dependencies discovered.' Reference any existing DR runbook in Confluence if available.
- **[warning]** 3. Module Reference - Clear Integration Endpoint - Configuration: The URL uses 'integrations.apsis.one' but the DSM webhook URL format uses 'integration.[env].apsis.cloud'. Inconsistent domain patterns across the document for Apsis-hosted services may confuse an agent constructing API calls.
  - Fix: Audit all service URLs in the document for consistency. Add a note clarifying which services are on which domain/subdomain pattern (e.g., public-facing vs. internal).
- **[warning]** 5. Business Rules Reference - Form Submissions: Rule 3 states 'Minimum identifying info required: CRM ID OR (email AND/OR phone)'. The AND/OR inside the parenthetical is ambiguous — does email alone suffice, phone alone suffice, or must both be present? This ambiguity could cause incorrect agent behavior when validating form submissions.
  - Fix: Clarify the exact logic, e.g., 'CRM ID alone is sufficient. Alternatively, email alone OR phone alone OR both together are each sufficient substitutes for CRM ID.'
- **[warning]** 3. Module Reference - Consent Management - Business rules: The Magento auto-mapping bug fix is described as 'remove AND NOT clause in conditional check; only check update_email == true' but this appears to be a proposed fix, not a deployed one. The document does not clearly state whether this fix has been applied.
  - Fix: Explicitly state current deployment status: 'Fix NOT yet deployed as of document generation date.' Add to Known Issues table with a status column if possible, or annotate with ⚠️.
- **[suggestion]** 8. Confidence Notes - Gaps / Unresolved: 'Golfamore/Intermail keyspace remapping solution (Option 3 vs Option 2) decision pending meeting (Jan 22, 2026 mentioned in transcript)' — the document was generated 2026-03-20, meaning this meeting date is 2 months in the past. The gap note is stale.
  - Fix: Update this gap note to reflect whether the Jan 22 meeting occurred and what decision was reached, or explicitly mark as 'Status unknown — verify with team.'
- **[suggestion]** 2. Architecture Map - CI/CD: CI/CD section mentions 'beta → beta (EU West only)' but does not explain what beta environment is used for or who has access. An agent asked to deploy to beta has no context for when this is appropriate.
  - Fix: Add a one-sentence description of beta environment purpose, e.g., 'Beta environment used for pre-production validation with selected customers; EU West only; requires explicit branch promotion.'
- **[suggestion]** 3. Module Reference - Retry Driver Lambda: The rule 'NEVER redrive consent opt-in messages' is critical but only stated once in the Retry Driver Lambda section. Given its severity (could cause double opt-in GDPR violations), this should be more prominent.
  - Fix: Add a prominent warning block (e.g., '⛔ CRITICAL:') to the Retry Driver Lambda section and ensure the cross-reference in Section 5 (Idempotency) and Section 4 (Error Handling) is equally emphatic.
- **[suggestion]** 7. Glossary: The glossary entry for 'Justin' states it is 'named after Justin Timberlake' — this is informal trivia. More importantly, it does not clarify the relationship between 'Justin' (branding) and the actual service/repo names an agent would encounter in code.
  - Fix: Add the actual repository name or service prefix used in code/AWS (e.g., 'Repository: justin-integrations or similar') so agents can correlate the brand name to technical artifacts.
- **[suggestion]** 3. Module Reference - Profile List Sync Manager: Tag name is listed as 'FEC Enterprise' in the example but other sections consistently use 'FSC Enterprise'. This may be a typo.
  - Fix: Verify the correct tag name format. If it is 'FSC Enterprise', correct the typo. If 'FEC Enterprise' is intentional for a specific connector, clarify.
- **[suggestion]** 3. Module Reference - Unified Data (Data Provider Pattern): The columnar format is described as 'Apache columnar format (Arrow/Parquet)' — Arrow and Parquet are distinct formats with different use cases (in-memory streaming vs. on-disk storage). For an agent implementing or debugging this integration, the ambiguity matters.
  - Fix: Specify which format is actually used: Apache Arrow (for streaming) or Parquet (for storage). Add a ⚠️ confidence flag if this is uncertain.