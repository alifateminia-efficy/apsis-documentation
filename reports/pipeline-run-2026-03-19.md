---
title: Pipeline Execution Report
run_date: 2026-03-19T21:21:00.815Z
status: COMPLETED_WITH_WARNINGS
---

# Pipeline Execution Report

## Run Summary

| Parameter | Value |
|-----------|-------|
| **Status** | COMPLETED_WITH_WARNINGS |
| **Run Date** | 2026-03-19T21:21:00.815Z |
| **Domain** | Integrations |
| **Source Repo** | efficy-sa/apsis-katie |
| **Source Branch** | master |
| **Source Folder** | /apsis-one/kt/integrations |
| **Destination Repo** | alifateminia-efficy/apsis-documentation |
| **Feature Branch** | docs/auto-2026-03-19-1773955232 |
| **Files Processed** | 4 |

## Models Used

| Phase | Model |
|-------|-------|
| Cleanup | claude-haiku-4-5-20251001 |
| Extraction | claude-haiku-4-5-20251001 |
| Synthesis | claude-haiku-4-5-20251001 |
| Self-Review | claude-haiku-4-5-20251001 |

## Token Usage

| Phase | Input Tokens | Output Tokens |
|-------|-------------|---------------|
| Cleanup (total) | 0 | 0 |
| Extraction (total) | 0 | 0 |
| Synthesis | 576 | 594 |
| Self-Review | 1,080 | 816 |
| **TOTAL** | **1,656** | **1,410** |

## Per-File Status

| File | Cleanup | Push | Extraction |
|------|---------|------|------------|
| Benjamin - Intergrations Overview.txt | ERROR | OK | PARSE_ERROR |
| Discuss display names for connectors PR changes.txt | ERROR | OK | PARSE_ERROR |
| Efficy Enterprise issues.txt | ERROR | OK | PARSE_ERROR |
| Erik - 20.11.2025.txt | ERROR | OK | PARSE_ERROR |

## Synthesis & Review

| Metric | Value |
|--------|-------|
| Synthesis truncated | No |
| Knowledge base pushed | Yes |
| Review result | needs_revision |
| Critical issues | 6 |
| Modules documented | ? |
| Business rules captured | ? |
| Known issues listed | ? |

## ⚠️ Extraction Parse Errors

- Benjamin - Intergrations Overview.txt
- Discuss display names for connectors PR changes.txt
- Efficy Enterprise issues.txt
- Erik - 20.11.2025.txt

## Review Issues

- **[critical]** Document Header & Metadata: Document is non-functional—all 4 KT sessions failed with identical parsing errors. Knowledge base contains zero domain content across 8 required sections.
  - Fix: Do not deploy this KB to production. Resolve runtime environment issues (Node.js fetch polyfill) and re-execute KT extraction before publishing.
- **[critical]** 1. Domain Overview: Section is empty with placeholder text 'UNABLE TO POPULATE'. No domain overview, architecture, or scope defined.
  - Fix: Populate with: domain purpose, scope boundaries, key entities, integration patterns supported, and stakeholder context.
- **[critical]** 3. Module Reference: No modules documented. Required for agent to understand codebase structure, dependencies, and function signatures.
  - Fix: Document all integration modules with: name, purpose, public APIs, dependencies, version info, and usage examples.
- **[critical]** 5. Business Rules Reference: No business rules captured. Agents cannot make correct decisions without understanding domain constraints.
  - Fix: Extract and document: validation rules, integration workflows, auth/security policies, rate limits, retry logic, and SLA requirements.
- **[critical]** 2. Architecture Map: Architecture diagram/description missing. Agents need to understand system topology and component relationships.
  - Fix: Provide: system diagram, component interactions, data flow, external integrations, and deployment topology.
- **[critical]** 4. Cross-Cutting Concerns: Section empty. Error handling, logging, monitoring, and security patterns undefined.
  - Fix: Document: error handling strategy, logging standards, monitoring/observability, security patterns, performance considerations, and testing approaches.
- **[warning]** 7. Glossary: Glossary is empty. Domain-specific terminology undefined.
  - Fix: Define integration-domain terms (e.g., webhook, payload, trigger, adapter, connector, sync, mapping, etc.).
- **[warning]** Generated Metadata: Future date in generation timestamp (2026-03-19). Timestamps should reflect actual generation date.
  - Fix: Correct timestamp to actual generation date and time.
- **[suggestion]** 6. Known Issues & Workarounds: Table documents only meta-issues (extraction failures) rather than actual domain-level known issues.
  - Fix: After KB is populated, replace with real integration domain issues: common failure modes, edge cases, version incompatibilities, deprecated features, etc.