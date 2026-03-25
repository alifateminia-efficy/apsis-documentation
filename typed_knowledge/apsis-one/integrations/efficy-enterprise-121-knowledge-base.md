---
title: Apsis One Integrations — efficy enterprise 12.1 Knowledge Base
subdomain: efficy enterprise 12.1
generated: 2026-03-25T12:40:46.853Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (1 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — efficy enterprise 12.1 Knowledge Base](#apsis-one-integrations-efficy-enterprise-121-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Installation Manager](#installation-manager)
    - [Form Sync Event Listener System](#form-sync-event-listener-system)
    - [Form Submission Routing](#form-submission-routing)
    - [Contact Keyspace (Entity-Specific Keyspace for Efficy 12.1)](#contact-keyspace-entity-specific-keyspace-for-efficy-121)
    - [CRM Data Download Handler](#crm-data-download-handler)
    - [Consent Export Logic](#consent-export-logic)
    - [Profile Merging Engine](#profile-merging-engine)
    - [Keyspace Discriminator Generation & Lookup](#keyspace-discriminator-generation-lookup)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling](#error-handling)
    - [Logging & Debugging](#logging-debugging)
    - [Deployment & Configuration](#deployment-configuration)
    - [Data Consistency & GDPR](#data-consistency-gdpr)
    - [Integration Uninstallation](#integration-uninstallation)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Contact Data Mastery & Sync Direction](#contact-data-mastery-sync-direction)
    - [Form Submission & Entity Routing](#form-submission-entity-routing)
    - [Keyspace Lifecycle & Discriminator Management](#keyspace-lifecycle-discriminator-management)
    - [Entity Type Constraints](#entity-type-constraints)
    - [Installation & Uninstallation Workflows](#installation-uninstallation-workflows)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Issue 1: No Efficy Enterprise 12.1-Specific Known Issues Documented](#issue-1-no-efficy-enterprise-121-specific-known-issues-documented)
    - [Issue 2: Keyspace Discriminator Hash Collision Risk (Cross-Platform, Affects Efficy 12.1)](#issue-2-keyspace-discriminator-hash-collision-risk-cross-platform-affects-efficy-121)
    - [Issue 3: Silent Failure on Keyspace Lookup](#issue-3-silent-failure-on-keyspace-lookup)
    - [Issue 4: No Retry Logic for Failed CRM Submissions](#issue-4-no-retry-logic-for-failed-crm-submissions)
    - [Issue 5: Uninstall Mid-Sync May Leave Partial State](#issue-5-uninstall-mid-sync-may-leave-partial-state)
    - [Issue 6: GDPR Deletion Workflow Not Documented](#issue-6-gdpr-deletion-workflow-not-documented)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence ✅](#high-confidence-)
    - [Medium Confidence ⚠️](#medium-confidence-)
    - [Low Confidence ❌](#low-confidence-)
    - [Gaps & Unresolved Questions 🔍](#gaps-unresolved-questions-)
    - [Outdated or Disputed Information](#outdated-or-disputed-information)
  - [9. Architecture Decision Log](#9-architecture-decision-log)
    - [Decision: Keyspace Discriminator Format (Non-Reversible Hash)](#decision-keyspace-discriminator-format-non-reversible-hash)
    - [Decision: Never Delete Keyspace on Integration Uninstall](#decision-never-delete-keyspace-on-integration-uninstall)
    - [Decision: Form 'sync to CRM' Option Visibility Conditional on Installation](#decision-form-sync-to-crm-option-visibility-conditional-on-installation)
    - [Decision: Contact-Only Entity Type for Efficy Enterprise 12.1](#decision-contact-only-entity-type-for-efficy-enterprise-121)
  - [10. Integration Procedures Reference](#10-integration-procedures-reference)
    - [Procedure: Install Efficy Enterprise 12.1 Integration](#procedure-install-efficy-enterprise-121-integration)
    - [Procedure: Submit Form with Efficy 12.1 Sync Enabled](#procedure-submit-form-with-efficy-121-sync-enabled)
    - [Procedure: Download Contacts from Efficy 12.1](#procedure-download-contacts-from-efficy-121)
    - [Procedure: Export Consent Changes to Efficy 12.1](#procedure-export-consent-changes-to-efficy-121)
    - [Procedure: Uninstall Efficy 12.1 Integration](#procedure-uninstall-efficy-121-integration)
    - [Procedure: Reinstall Efficy 12.1 Integration (Reuse Keyspace)](#procedure-reinstall-efficy-121-integration-reuse-keyspace)
    - [Procedure: GDPR Delete Contact from Efficy 12.1 (Manual)](#procedure-gdpr-delete-contact-from-efficy-121-manual)
  - [11. Deployment & Operations Guide](#11-deployment-operations-guide)
    - [Environment Configuration (To Be Documented)](#environment-configuration-to-be-documented)
- [Efficy Enterprise 12.1 Integration](#efficy-enterprise-121-integration)
- [Keyspace Configuration](#keyspace-configuration)
- [Form Sync Configuration](#form-sync-configuration)
- [Error Handling](#error-handling)
    - [Monitoring & Alerting (To Be Implemented)](#monitoring-alerting-to-be-implemented)

---

# Apsis One Integrations — efficy enterprise 12.1 Knowledge Base

## 1. Subdomain Overview

The **efficy enterprise 12.1** subdomain manages bidirectional synchronization between Apsis One and Efficy Enterprise 12.1 CRM installations. It handles contact data flow, form submission routing, keyspace management for multi-installation scenarios, and consent export back to the CRM. Efficy Enterprise 12.1 operates as the master data source for contact records and supports only the contact entity type (no leads or silhouettes). This subdomain sits within the broader Apsis One Integrations platform, which orchestrates similar workflows for multiple CRM systems (E-deal, Microsoft Dynamics, Tribe, JoinCX, etc.).

---

## 2. Architecture Map

```
┌─────────────────────────────────────────────────────────────┐
│  Efficy Enterprise 12.1 CRM System (Master Data Source)      │
│  - Contact records only                                      │
│  - Form submission responses (entity type: 'contact')        │
│  - Data download/sync endpoints                              │
│  - Consent export webhooks                                   │
└────────────┬────────────────────────────────────────────────┘
             │ REST API (bidirectional)
             │
┌────────────▼────────────────────────────────────────────────┐
│  Apsis One Integrations Platform                             │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Installation Manager                                  │   │
│  │ └─> Keyspace Discriminator Generation                │   │
│  │     (integrations:keyspaces:[8-char-hash]:           │   │
│  │      [CRM-logical-name])                             │   │
│  └──────────────────────────────────────────────────────┘   │
│           │                                                  │
│           ├─> Create Main Contact Keyspace                  │
│           │   (no leads/silhouettes for Efficy 12.1)        │
│           │                                                  │
│           └─> Register Event Listeners                      │
│               (form 'start viewed', 'submit' events)        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Form Sync Event Listener System                       │   │
│  │ - Listens for form submissions (if sync enabled)      │   │
│  │ - Routes to Form Submission Handler                   │   │
│  └──────────────────────────────────────────────────────┘   │
│           │                                                  │
│           └─> Form Submission Routing                       │
│               ├─> Collect: profileKey, email, phone,        │
│               │   existing CRM ID, form fields              │
│               ├─> Send to Efficy Enterprise 12.1            │
│               ├─> Receive: entityType='contact', recordId   │
│               └─> Store CRM ID in Contact Keyspace          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Keyspace System (Database)                            │   │
│  │ ┌────────────────────────────────────────────────┐   │   │
│  │ │ Contact Keyspace (entity type: 'contact')      │   │   │
│  │ │ Discriminator: integrations:keyspaces:         │   │   │
│  │ │ [8-char-hash]:efficy-enterprise-12-1           │   │   │
│  │ │ - Profile keys → CRM record IDs                │   │   │
│  │ │ - Event history (form submits, opens, etc.)    │   │   │
│  │ │ - Consent state                                │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  │ ┌────────────────────────────────────────────────┐   │   │
│  │ │ Discriminator Lookup Table                      │   │   │
│  │ │ (sectionId, crmType, entityType)               │   │   │
│  │ │  → discriminator, keyspaceId                   │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│           │                                                  │
│           └─> CRM Data Download Handler                     │
│               ├─> Poll/webhook from Efficy 12.1             │
│               ├─> For each contact returned:                │
│               │   ├─> Lookup keyspace by entityType         │
│               │   └─> Update profile in keyspace            │
│               └─> Merge silhouettes (N/A for Efficy 12.1)   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Consent Export Logic                                 │   │
│  │ - Query Contact Keyspace for consent changes         │   │
│  │ - Exclude silhouettes/leads (none for Efficy 12.1)   │   │
│  │ - Send only contact consent to Efficy 12.1           │   │
│  └──────────────────────────────────────────────────────┘   │
│           │                                                  │
└───────────┼──────────────────────────────────────────────────┘
            │ REST API
            │ (consent export payload)
            │
┌───────────▼──────────────────────────────────────────────────┐
│  Efficy Enterprise 12.1 CRM (Receives consent updates)        │
└───────────────────────────────────────────────────────────────┘
```

**Key relationships:**
- **Installation Manager** ↔ **Keyspace System**: bootstraps discriminator and creates contact keyspace
- **Form Sync Listener** ↔ **Form Submission Routing**: routes form data to CRM, receives contact ID back
- **Keyspace System** ↔ **Data Download Handler**: stores downloaded contact records by keyspace lookup
- **Consent Export Logic** ↔ **Efficy Enterprise 12.1**: sends consent changes back to CRM
- **Profile Merging Engine**: N/A for Efficy 12.1 (no lead/silhouette entity types)

---

## 3. Module Reference

### Installation Manager
- **Purpose**: Bootstraps keyspace discriminator and creates contact keyspace when Efficy Enterprise 12.1 integration is installed on an account section.
- **Key files**: `Installation Manager (main_install function)` [exact path not provided in transcript]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: CRM system type (efficy-enterprise-12-1), account section ID, installation request
  - **Out**: Discriminator string stored in database, keyspace ID created in keyspace DB, event listeners registered
- **Business rules**:
  - Never delete keyspace on uninstall; reuse on reinstall
  - Efficy 12.1 supports only contact entity type; no lead/silhouette keyspaces created
  - Discriminator format: `integrations:keyspaces:[8-char-hash]:efficy-enterprise-12-1`
  - 8-char hash derived from section discriminator (rationale for 8 chars not fully documented)
- **Configuration**:
  - CRM logical name: `efficy-enterprise-12-1`
  - Entity types for Efficy 12.1: `[contact]` (hardcoded per CRM type)
  - Keyspace reuse logic: check database for existing discriminator before creating new keyspace
- **Integration points**:
  - **Efficy Enterprise 12.1**: none during installation (metadata assumed hardcoded)
  - **Internal Keyspace DB**: store/retrieve discriminator mappings
- **Gotchas**:
  - Reinstalling after uninstall does NOT create new keyspace; lookup fails silently if discriminator mapping is deleted
  - Entity type configuration is hardcoded per CRM; cannot be customized per customer installation
  - No automated validation that discriminator hash is unique; collision risk exists but is low
- **Tribal knowledge**:
  - The 8-character hash is for uniqueness only; no reverse resolution needed. This is intentional.
  - Uninstalling integration does NOT delete keyspace—this preserves historical event data for re-engagement campaigns (critical for "win-back" customer scenarios).

---

### Form Sync Event Listener System
- **Purpose**: Registers event listeners on form campaigns when 'sync to CRM' option is enabled, triggering CRM submission on form completion.
- **Key files**: Event listener framework (exact path not provided)
- **Tech stack**: Apsis One Integrations event listener framework
- **Data flow**:
  - **In**: Form campaign published with 'sync to CRM' enabled, event type (start viewed, submit)
  - **Out**: Listener registration stored, submit event fires → Form Submission Routing triggered
- **Business rules**:
  - 'sync to CRM' option only visible/available if integration is installed
  - Listeners registered for 'start viewed' and 'submit' events only
  - Listeners removed immediately upon integration uninstallation
  - Form submission will not reach CRM under normal circumstances if integration not installed
- **Configuration**:
  - Event types: `['start viewed', 'submit']`
  - Visibility condition: integration installed check
  - Listener lifecycle: install registration, uninstall cleanup
- **Integration points**:
  - **Form Submission Routing**: receives submit event and passes form data
  - **Efficy Enterprise 12.1**: indirectly via Form Submission Routing
- **Gotchas**:
  - ⚠️ If 'sync to CRM' option is not explicitly enabled, form submissions do NOT reach Efficy 12.1 (even if integration is installed)
  - Listeners removed on uninstall, but no data cleanup occurs in keyspace
  - Hidden UI option prevents accidental form routing to inactive integrations
- **Tribal knowledge**:
  - The 'sync to CRM' option won't even appear if integration isn't installed. This is a safety feature preventing orphaned form submissions.

---

### Form Submission Routing
- **Purpose**: Routes form submissions to Efficy Enterprise 12.1, receives contact record ID, and stores ID in contact keyspace.
- **Key files**: [exact path not provided]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Form submit event, profileKey (from Apsis), identifying data (email, phone, existing CRM ID), form field values
  - **Out**: HTTP request to Efficy 12.1 endpoint with payload; receives entityType='contact' + recordId; stores recordId in contact keyspace
- **Business rules**:
  - Form payload must include profile key (added by Apsis) and identifying data (email or phone required)
  - Efficy 12.1 always responds with entityType='contact' (no leads/silhouettes)
  - If contact already exists in Efficy 12.1, received ID is existing CRM ID; new contacts receive new ID
  - CRM ID stored in contact keyspace immediately upon receipt
- **Configuration**:
  - Form submission payload structure: `{ profileKey, identifyingData: {email, phone, crmId}, fields: {...} }`
  - Expected CRM response: `{ entityType: 'contact', recordId: <string> }`
  - Keyspace lookup: use entityType='contact' to retrieve discriminator from database, then keyspace ID
- **Integration points**:
  - **Efficy Enterprise 12.1 REST API**: form submission endpoint (exact URL not provided)
  - **Contact Keyspace**: stores returned recordId
  - **Event Listener System**: receives submit event trigger
- **Gotchas**:
  - ⚠️ Efficy 12.1 entity type is always 'contact'; if code mistakenly allows silhouette/lead storage, consent export logic will break
  - If keyspace discriminator lookup fails, submission may proceed without keyspace storage (silent failure risk)
  - Existing CRM ID in identifying data is optional; if missing, Efficy 12.1 creates new contact
  - No retry logic documented; timeout/failure handling unclear
- **Tribal knowledge**:
  - Unlike E-deal (which returns 'silhouette') or Tribe (which returns dynamic entity type), Efficy 12.1 only ever returns 'contact'. This simplifies routing compared to other CRM systems.

---

### Contact Keyspace (Entity-Specific Keyspace for Efficy 12.1)
- **Purpose**: Stores profile key ↔ Efficy Enterprise 12.1 contact ID mappings and event history for Efficy 12.1 contact records only.
- **Key files**: Keyspace DB (exact schema not provided)
- **Tech stack**: Apsis One Integrations keyspace storage (appears to be NoSQL or document DB)
- **Data flow**:
  - **In**: Form submission → recordId from Efficy 12.1; data download → contact records from Efficy 12.1
  - **Out**: Keyspace ID → queried by consent export logic; profile data → queried for merge operations (N/A for Efficy 12.1)
- **Business rules**:
  - Discriminator format: `integrations:keyspaces:[8-char-hash]:efficy-enterprise-12-1`
  - One contact keyspace created per Efficy 12.1 installation per account section
  - No lead or silhouette keyspaces created (Efficy 12.1 supports contacts only)
  - Profile key ↔ CRM ID mapping is 1:1 (assuming unique CRM IDs per contact)
  - Historical event data (form submits, opens, clicks, etc.) preserved in keyspace for re-engagement campaigns
  - Keyspace never deleted on integration uninstall; reused on reinstall to preserve history
- **Configuration**:
  - Discriminator: hardcoded format with section-derived 8-char hash + CRM logical name
  - Keyspace creation: triggered by Installation Manager
  - Lookup table: (sectionId, crmType='efficy-enterprise-12-1', entityType='contact') → discriminator → keyspaceId
- **Integration points**:
  - **Installation Manager**: creates keyspace + stores discriminator
  - **Form Submission Routing**: stores recordId in keyspace
  - **CRM Data Download Handler**: updates contact records in keyspace
  - **Consent Export Logic**: queries keyspace for consent changes
- **Gotchas**:
  - ⚠️ Keyspace reuse on reinstall means old data is never deleted; if customer re-engages with same email after uninstall, old activity history is visible
  - If discriminator lookup fails during form submission, recordId may not be stored in keyspace (silent failure)
  - No documented cleanup for orphaned or duplicate profile keys in keyspace
  - Profile key collision risk not mentioned but theoretically possible across sections
- **Tribal knowledge**:
  - The keyspace persists across uninstall/reinstall cycles intentionally. This enables "win-back" scenarios where you want to re-engage customers with their full event history visible.
  - Efficy 12.1 has no concept of leads internally; all temporary contacts are handled in the CRM, not in Apsis.

---

### CRM Data Download Handler
- **Purpose**: Polls or receives webhook events from Efficy Enterprise 12.1 for contact data syncs; updates contact records in contact keyspace.
- **Key files**: [exact path not provided]
- **Tech stack**: Apsis One Integrations platform (webhook or polling mechanism not specified)
- **Data flow**:
  - **In**: Contact records from Efficy 12.1 (via webhook or polling), entity type='contact'
  - **Out**: Keyspace lookup by entityType → fetch discriminator + keyspaceId; update contact record in keyspace
- **Business rules**:
  - Only entity type returned is 'contact' (no leads/silhouettes)
  - For each contact record, look up keyspace discriminator using entityType='contact'
  - Update existing profile in keyspace or create new record if CRM ID not previously synced
  - No merge operations triggered (N/A for Efficy 12.1)
- **Configuration**:
  - Sync frequency: not specified (varies by customer configuration)
  - Webhook endpoint: not provided
  - Polling interval: not provided
  - Payload structure: not fully documented
- **Integration points**:
  - **Efficy Enterprise 12.1**: contact download endpoint or webhook
  - **Contact Keyspace**: update records by keyspace lookup
- **Gotchas**:
  - ⚠️ If keyspace discriminator lookup fails, sync may fail silently without updating keyspace
  - Entity type in Efficy 12.1 response is always 'contact'; no error handling for unexpected types
  - No documented deduplication logic if Efficy 12.1 sends duplicate contacts in single sync
  - Sync failure/timeout handling not documented
- **Tribal knowledge**:
  - Efficy 12.1 downloads are straightforward because there's only one entity type (contact). Other CRMs require more complex routing logic.

---

### Consent Export Logic
- **Purpose**: Sends contact consent changes back to Efficy Enterprise 12.1, excluding any non-contact records (silhouettes/leads, which don't exist for Efficy 12.1).
- **Key files**: [exact path not provided]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Consent change events from Apsis (email opt-in/opt-out, SMS opt-in/opt-out, etc.), contact keyspace query
  - **Out**: HTTP request to Efficy 12.1 consent endpoint with contact CRM IDs and consent state
- **Business rules**:
  - Query contact keyspace only (no silhouettes/leads to filter out for Efficy 12.1)
  - Send only contact records to Efficy 12.1
  - CRM ID ↔ Apsis profile key mapping retrieved from keyspace
  - Consent state includes all consent fields (email, SMS, postal, etc., if applicable)
  - Never include non-contact records in export payload (critical: silhouettes in contact keyspace would be exported incorrectly)
- **Configuration**:
  - Consent export endpoint: Efficy 12.1 API (exact URL not provided)
  - Payload format: (contactId, consentState: {...}) [exact format not provided]
  - Trigger: consent change event or batch export schedule
- **Integration points**:
  - **Contact Keyspace**: query for profile → CRM ID mappings
  - **Efficy Enterprise 12.1 Consent API**: send consent changes
- **Gotchas**:
  - ⚠️ **CRITICAL for multi-entity CRM systems**: If silhouettes/leads were mistakenly stored in contact keyspace (e.g., in E-deal), consent export would include them, causing data corruption in CRM. Efficy 12.1 only has contacts, but same principle applies across integration platform.
  - If keyspace lookup fails, consent changes may not be sent to Efficy 12.1
  - No documented retry logic for failed exports
  - Batch export frequency/timing not specified
- **Tribal knowledge**:
  - This is why entity keyspace separation is so critical. Storing a silhouette ID in the contact keyspace would cause the consent export logic to send silhouette data back to the CRM, breaking the CRM data model. Efficy 12.1 doesn't have silhouettes, but this principle is why the architecture exists.

---

### Profile Merging Engine
- **Purpose**: Merges records between entity keyspaces (lead ↔ contact); N/A for Efficy Enterprise 12.1 since only contact entity exists.
- **Key files**: [exact path not provided]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Merge request from CRM or form response indicating multiple entity types; silhouette ID + profile key
  - **Out**: Merged profile in keyspace (two IDs now map to same profile to prevent duplicates)
- **Business rules**:
  - Merge only within integration entity keyspaces (lead ↔ contact, silhouette ↔ contact, etc.)
  - **NEVER merge CRM-originated contacts with external keyspaces** (e.g., email keyspace). Such merges won't propagate back to CRM and will cause data inconsistency.
  - Merge only occurs when CRM explicitly requests it or when form submission returns multiple entity types
  - After merge, duplicate form submissions with same email will identify existing merged profile instead of creating new record
  - N/A for Efficy 12.1 (no leads/silhouettes), but mechanism applies to other CRM systems in platform
- **Configuration**:
  - Merge trigger: form response with multiple entity types OR explicit merge request from CRM
  - Merge type: silhouette → contact (or lead → contact in multi-entity systems)
- **Integration points**:
  - **Entity Keyspaces**: retrieves source + destination keyspace records
  - **Form Submission Routing**: receives merge signal from form response
  - **Efficy Enterprise 12.1**: none (Efficy 12.1 doesn't support leads/silhouettes)
- **Gotchas**:
  - ⚠️ Merges initiated in Apsis UI (not via CRM) will NOT propagate back to CRM, causing inconsistency. CRM must be master of merge decisions.
  - If merge fails silently, duplicate profiles may be created in subsequent form submissions
  - No documented prevention of circular merges (profile A ↔ B ↔ C ↔ A)
- **Tribal knowledge**:
  - CRM is the master of contact data. If two contacts need to be merged, merge them in Efficy 12.1 first, and the merge will flow to Apsis. Don't merge in Apsis and expect it to propagate back to Efficy 12.1—it won't.

---

### Keyspace Discriminator Generation & Lookup
- **Purpose**: Generates unique discriminator strings for each keyspace and provides lookup mechanism to retrieve keyspace ID from entity type.
- **Key files**: Installation Manager (discriminator generation), Lookup table (discriminator retrieval)
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In (generation)**: Account section ID, CRM logical name, entity type
  - **Out (generation)**: Discriminator string → stored in database
  - **In (lookup)**: Entity type (e.g., 'contact')
  - **Out (lookup)**: Keyspace ID retrieved from database
- **Business rules**:
  - Discriminator format: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`
  - 8-char hash derived from section discriminator (hash algorithm/source not documented)
  - One discriminator per (sectionId, crmType, entityType) tuple
  - Discriminator lookup is one-way (hash not reverse-resolvable by design)
  - Case sensitivity of hash not specified
- **Configuration**:
  - Hash length: 8 characters (rationale for 8 not fully documented; collision risk low)
  - CRM logical name for Efficy 12.1: `efficy-enterprise-12-1`
  - Database table: (sectionId, crmType, entityType, discriminator, keyspaceId)
- **Integration points**:
  - **Installation Manager**: generates discriminator on install
  - **Form Submission Routing**: looks up keyspace ID by entity type
  - **CRM Data Download Handler**: looks up keyspace ID by entity type
  - **Consent Export Logic**: queries keyspace by discriminator-derived keyspace ID
- **Gotchas**:
  - ⚠️ No reverse resolution mechanism. If two sections have hash collision (unlikely but theoretically possible), no way to identify source section.
  - If discriminator lookup fails in database, subsequent operations fail silently
  - Discriminator format assumes entity type is always present (but Efficy 12.1 only has 'contact', so implicit)
  - Database query performance not documented; lookup may be slow with large discriminator tables
- **Tribal knowledge**:
  - The hash is for uniqueness only. You don't need to reverse-resolve it. This is intentional to prevent coupling between sections.
  - 8 characters was chosen but why 8 specifically is lost tribal knowledge—current team assumes it's good enough.

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **Status**: Not specified in transcript
- **Relevance**: Form submissions and data downloads require auth to Efficy Enterprise 12.1 endpoint
- **⚠️ Gap**: Auth mechanism (API key, OAuth, Basic Auth) not documented for this subdomain

### Error Handling
- **Silent failures**: Keyspace lookup failures, CRM API timeouts, consent export failures may occur without logging/alerting
- **No retry logic documented** for form submissions or data downloads
- **Form submission failure**: unclear if form is rejected in Apsis or submission is lost silently
- **Recommendation**: Implement exponential backoff + DLQ for failed CRM submissions

### Logging & Debugging
- **No logging strategy documented**
- **Recommendation**: Log all keyspace discriminator lookups, form submissions to Efficy 12.1, and consent exports
- **Tracing**: Form submission ID should be tracked end-to-end (Apsis → Efficy 12.1 → keyspace update)

### Deployment & Configuration
- **CRM logical name hardcoding**: `efficy-enterprise-12-1` appears hardcoded in Installation Manager
- **Entity type configuration**: Contact-only entity list hardcoded per CRM
- **⚠️ Gap**: No environment-based configuration documented (dev/staging/prod endpoints for Efficy 12.1)
- **Recommendation**: Externalize Efficy 12.1 endpoint URL and API key to config file/env vars

### Data Consistency & GDPR
- **GDPR compliance**: Keyspaces never deleted on uninstall, but GDPR cleanup requests should hard-delete keyspace records
- **Consent state tracking**: Consent export must be idempotent (resend doesn't create duplicates)
- **⚠️ Gap**: No documented GDPR deletion workflow for Efficy 12.1 contact records

### Integration Uninstallation
- **Keyspace reuse**: Existing keyspace reused on reinstall (by design)
- **Event listeners**: Removed from form campaigns on uninstall
- **⚠️ Gap**: No documented cleanup of partially-synced form submissions if uninstall occurs mid-sync

---

## 5. Business Rules Reference

### Contact Data Mastery & Sync Direction
| Rule | Context | Enforcement |
|------|---------|-------------|
| **Efficy 12.1 is master of contact data** | All contact records originate in Efficy 12.1 | CRM ID stored in Apsis keyspace; changes in Efficy 12.1 flow to Apsis via download sync |
| **Profiles synced to Apsis remain unless explicitly deleted** | GDPR compliance, re-engagement scenarios | Keyspace never deleted on uninstall; manual GDPR request required to hard-delete |
| **Never merge CRM-originated contacts in Apsis** | Data consistency enforcement | Merges initiated in Apsis won't propagate back to Efficy 12.1 |
| **Consent changes must sync back to Efficy 12.1** | CRM consent state must match Apsis | Consent export logic triggered on any consent change event |

### Form Submission & Entity Routing
| Rule | Context | Enforcement |
|------|---------|-------------|
| **'sync to CRM' option only visible if integration installed** | Prevent orphaned submissions | Form campaign UI hides option if integration not installed |
| **Form submission requires profile key + identifying data** | Identify contact in Apsis | Email or phone mandatory in form payload |
| **Efficy 12.1 always responds with entityType='contact'** | Entity routing | No lead/silhouette handling needed for this CRM |
| **CRM ID stored in contact keyspace immediately upon receipt** | Enable re-engagement and duplicate prevention | Form Submission Routing stores ID before returning to form UI |

### Keyspace Lifecycle & Discriminator Management
| Rule | Context | Enforcement |
|------|---------|-------------|
| **One discriminator per (section, CRM type, entity type) tuple** | Uniqueness for multi-installation scenarios | Installation Manager generates and stores discriminator |
| **Discriminator format: integrations:keyspaces:[8-char-hash]:[efficy-enterprise-12-1]** | Standardized uniqueness | Hardcoded in discriminator generation logic |
| **Keyspace never deleted on uninstall; reused on reinstall** | Historical data preservation for re-engagement | Installation Manager looks up existing keyspace before creating new |
| **Hash component is for uniqueness only (non-reversible)** | Decouple section identity from discriminator | No reverse-resolution mechanism; hash not inspected at runtime |

### Entity Type Constraints
| Rule | Context | Enforcement |
|------|---------|-------------|
| **Efficy 12.1 supports contact entity type only** | CRM data model constraint | Entity type list hardcoded in Installation Manager: [contact] |
| **No lead or silhouette keyspaces created for Efficy 12.1** | Prevent data model conflicts | Installation Manager loop skips non-contact entities |
| **Consent export filters by entity type (contact only)** | Prevent incorrect exports | Consent Export Logic queries contact keyspace exclusively |

### Installation & Uninstallation Workflows
| Rule | Context | Enforcement |
|------|---------|-------------|
| **Installation creates discriminator + contact keyspace + event listeners** | Bootstrap integration | Installation Manager orchestrates all three steps |
| **Uninstallation removes event listeners but NOT keyspace** | Preserve historical data; simplify reinstall | Form Sync Event Listener System deregisters listeners; keyspace remains in DB |
| **Reinstallation reuses existing keyspace + discriminator** | Enable re-engagement with full history | Installation Manager looks up discriminator before creating new |
| **Form event listeners removed on uninstall** | Prevent orphaned submissions after uninstall | Event Listener System deregisters 'start viewed' and 'submit' listeners |

---

## 6. Known Issues & Workarounds

### Issue 1: No Efficy Enterprise 12.1-Specific Known Issues Documented
**Severity**: Low  
**Context**: Efficy Enterprise 12.1 is simpler than other CRM systems (contact-only), so fewer edge cases emerge.  
**Status**: No issues reported in transcript specific to Efficy 12.1.  
**Workaround**: N/A

### Issue 2: Keyspace Discriminator Hash Collision Risk (Cross-Platform, Affects Efficy 12.1)
**Severity**: Low  
**Context**: 8-character hash may theoretically collide across different sections, though risk is low.  
**Impact**: If collision occurs, two installations on same account would map to same keyspace, corrupting contact data.  
**Status**: Unresolved; no mitigation documented.  
**Workaround**: Collision risk assumed acceptable due to low probability (8 chars ≈ 4.3 billion combinations). No reverse resolution mechanism to identify source section if collision suspected.

### Issue 3: Silent Failure on Keyspace Lookup
**Severity**: Medium  
**Context**: If discriminator lookup fails during form submission or data download, no error is logged.  
**Impact**: Form submission proceeds without storing CRM ID in keyspace, or contact download skips keyspace update.  
**Status**: Unresolved.  
**Workaround**: Implement explicit lookup error handling with logging and alerting.

### Issue 4: No Retry Logic for Failed CRM Submissions
**Severity**: Medium  
**Context**: Form submissions may fail due to Efficy 12.1 API timeout or temporary unavailability.  
**Impact**: Form submission lost; contact not created in Efficy 12.1.  
**Status**: Unresolved.  
**Workaround**: Implement dead-letter queue (DLQ) for failed submissions; manual retry mechanism.

### Issue 5: Uninstall Mid-Sync May Leave Partial State
**Severity**: Low  
**Context**: If integration is uninstalled while form submission is in-flight to Efficy 12.1, event listeners are removed but form submission may complete.  
**Impact**: Contact created in Efficy 12.1 but not stored in Apsis keyspace (listeners removed).  
**Status**: Unresolved.  
**Workaround**: Ensure uninstall blocks pending form submissions or implements graceful shutdown of event listeners.

### Issue 6: GDPR Deletion Workflow Not Documented
**Severity**: Medium  
**Context**: Keyspaces are never auto-deleted on uninstall; GDPR requests require explicit manual deletion or automated workflow.  
**Impact**: Customer data remains in Apsis indefinitely unless GDPR request is processed.  
**Status**: Unresolved.  
**Workaround**: Implement explicit GDPR deletion API that hard-deletes keyspace records by account/section ID.

---

## 7. Glossary

| Term | Definition | Efficy 12.1 Context |
|------|-----------|-------------------|
| **Keyspace** | Unique identifier and storage container for a specific entity type within a specific CRM installation. Identified by discriminator. | Contact keyspace only; stores profile key ↔ contact ID mappings and event history for Efficy 12.1 contacts. |
| **Discriminator** | Unique identifier for a keyspace: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`. Used for database lookup to retrieve keyspace ID. | Example: `integrations:keyspaces:a1b2c3d4:efficy-enterprise-12-1`. Lookup returns contact keyspace ID. |
| **CRM ID / Record ID** | Unique identifier for a contact in Efficy Enterprise 12.1. Returned in form submission response and data downloads. Stored in contact keyspace. | Maps 1:1 to Apsis profile key within contact keyspace. |
| **Profile Key** | Unique identifier for an Apsis profile. Added to form submission payloads and stored in keyspace alongside CRM ID. | Used to link form submission with Apsis profile; enables event history tracking in keyspace. |
| **Entity Type** | Classification of record type in CRM (contact, lead, silhouette, etc.). Used for keyspace routing. | Efficy 12.1: always 'contact' (no leads or silhouettes). |
| **Form Sync / 'sync to CRM'** | Feature enabling form submissions to be routed to integrated CRM. Must be explicitly enabled per form. | Form 'sync to CRM' option only visible if Efficy 12.1 integration installed. Submission payload sent to Efficy 12.1 endpoint. |
| **Event Listener** | Mechanism registering for form events (start viewed, submit) when sync enabled. Triggers CRM submission. Removed on uninstall. | Listeners registered during installation; fire on form submit to trigger Efficy 12.1 submission routing. |
| **Main Keyspace** | Primary keyspace for a CRM installation (typically contact entity). | For Efficy 12.1: contact keyspace is the only keyspace created. |
| **Entity Keyspace** | Keyspace dedicated to a specific entity type beyond main keyspace (leads, silhouettes, etc.). | N/A for Efficy 12.1 (no lead/silhouette entity keyspaces). |
| **CRM is Master** | Design principle: CRM system is authoritative source of contact data. Apsis should not override. | Efficy 12.1 is master for all contact records. Form submissions and data downloads ensure Apsis data matches Efficy 12.1. |
| **Consent Export** | Process sending contact consent changes back to CRM. Filters by entity type to exclude non-contacts. | Consent Export Logic queries contact keyspace, sends contact-only consent changes back to Efficy 12.1. |
| **Keyspace Reuse** | Existing keyspace is reused on reinstallation (not deleted on uninstall). | Preserves historical event data for re-engagement campaigns. Keyspace lookup by discriminator retrieves existing ID instead of creating new. |
| **Merge Request** | Request from CRM to merge two related entity records (lead → contact, silhouette → contact, etc.). | N/A for Efficy 12.1 (no leads/silhouettes to merge). |
| **Section Discriminator** | Unique identifier for an account section. Used as source for 8-char hash in keyspace discriminator. | Hashed to generate 8-char component of keyspace discriminator; hash algorithm/function not documented. |
| **Silent Failure** | Operation fails without logging or error notification (e.g., keyspace lookup fails, submission not stored). | Keyspace lookup failures during form routing or data downloads may silently skip keyspace update. |

---

## 8. Confidence Notes

### High Confidence ✅
- **Keyspace discriminator format & uniqueness principle**: Architecture decision well-documented with clear rationale
- **Efficy Enterprise 12.1 supports contact entity type only**: Confirmed no leads/silhouettes in this CRM
- **Keyspace reuse on reinstall**: Design decision explicitly stated; rationale clear (historical data preservation for re-engagement)
- **Form 'sync to CRM' option only visible if integration installed**: Safety feature with clear enforcement
- **CRM is master of contact data**: Foundational principle across platform; strongly enforced

### Medium Confidence ⚠️
- **8-character hash selection**: Stated as chosen but rationale not fully documented. Alternatives considered not detailed.
- **Discriminator lookup failure handling**: Appears to be silent failure, but error handling logic not fully specified
- **Efficy 12.1 endpoint authentication mechanism**: Not documented; assumed to be API key or OAuth but not confirmed
- **Entity type routing in form submission**: Logic inferred from description; exact code path not provided
- **Keyspace database schema**: Assumed to be (sectionId, crmType, entityType) → (discriminator, keyspaceId), but actual schema not provided

### Low Confidence ❌
- **Installation Manager exact file path**: Stated as "Installation Manager (main_install function)" but exact file path not provided
- **Form submission payload exact structure**: Format inferred; nested object structure and field names not confirmed
- **Efficy 12.1 response payload format**: Assumed to be `{entityType: 'contact', recordId}`, but exact structure not documented
- **Consent export payload format**: Assumed to be (contactId, consentState), but exact structure not provided
- **Data download sync trigger mechanism**: Unclear if webhook, polling, or event-driven; frequency not specified
- **Form listener event types**: Stated as 'start viewed' and 'submit', but completeness not confirmed

### Gaps & Unresolved Questions 🔍

#### Questions for Efficy Enterprise 12.1 Specialist
1. **What is the exact REST endpoint URL for form submissions to Efficy 12.1?** (e.g., `https://api.efficy.com/form-submit`)
2. **What authentication mechanism is used for Efficy 12.1 API calls?** (API key, OAuth 2.0, Basic Auth, etc.)
3. **What is the exact response payload structure for form submissions?** (Is entityType always included? Other fields?)
4. **Does Efficy 12.1 support contact updates via API, or only creation?**
5. **What is the sync frequency for contact downloads from Efficy 12.1?** (Real-time webhook, hourly batch, on-demand polling?)
6. **What is the exact consent export endpoint and payload format for Efficy 12.1?**
7. **How are contact deduplication and upserts handled?** (Email as unique key? CRM ID as primary key?)
8. **Does Efficy 12.1 return any error codes or status on form submission failure?** (For retry logic)
9. **Are there any field mapping rules between Apsis form fields and Efficy 12.1 contact fields?**
10. **How should GDPR deletions be handled?** (Delete in Apsis only, delete in Efficy 12.1, both?)

#### Questions for Platform Architecture
1. **Why specifically 8 characters for the keyspace discriminator hash?** (Performance, collision risk, other considerations?)
2. **What is the hash algorithm used to generate the 8-char hash from section discriminator?**
3. **Is the discriminator case-sensitive?**
4. **What is the exact database schema for discriminator mappings?**
5. **What error handling and retry logic exists for CRM API failures?**
6. **How are form submissions logged/traced end-to-end?**
7. **What is the deduplication strategy for Efficy 12.1 contacts in keyspace?**
8. **Is there a DLQ or async queue for failed CRM submissions?**

### Outdated or Disputed Information
- None identified in transcript; all information appears current as of KT session date (2026-03-25)

---

## 9. Architecture Decision Log

### Decision: Keyspace Discriminator Format (Non-Reversible Hash)
- **Status**: Accepted
- **Rationale**: Ensures uniqueness across multiple Efficy 12.1 installations on same account section without requiring reverse resolution. Hash component provides uniqueness without coupling to section identity.
- **Alternatives Rejected**: Fully qualified section ID (too verbose); sequential incrementing ID (poor scalability)
- **Impact**: Cannot identify source section from discriminator alone; must query database for full context

### Decision: Never Delete Keyspace on Integration Uninstall
- **Status**: Accepted
- **Rationale**: Preserve historical event data for customer re-engagement campaigns. Enable "win-back" scenarios. GDPR compliance: profiles remain in Apsis unless explicitly deleted.
- **Alternatives Rejected**: Delete keyspace on uninstall (loses historical data, breaks re-engagement)
- **Impact**: Keyspace persists indefinitely unless manually deleted via GDPR workflow

### Decision: Form 'sync to CRM' Option Visibility Conditional on Installation
- **Status**: Accepted
- **Rationale**: Prevent accidental form routing to inactive/uninstalled integrations
- **Alternatives Rejected**: Always show option, handle in submission logic (less safe)
- **Impact**: Form campaigns must be reconfigured if integration is uninstalled and reinstalled

### Decision: Contact-Only Entity Type for Efficy Enterprise 12.1
- **Status**: Accepted (CRM Constraint)
- **Rationale**: Efficy Enterprise 12.1 data model supports contacts only; no leads or silhouettes
- **Alternatives Rejected**: N/A (CRM-driven)
- **Impact**: Simpler routing logic; no multi-entity merge operations needed

---

## 10. Integration Procedures Reference

### Procedure: Install Efficy Enterprise 12.1 Integration
```
1. Activation trigger: Customer enables Efficy 12.1 integration on account section
2. Installation Manager.main_install() called with:
   - sectionId: <account section ID>
   - crmType: 'efficy-enterprise-12-1'
   - crmConfig: <Efficy 12.1 API endpoint, auth credentials>
3. Discriminator generation:
   - Extract 8-char hash from section discriminator
   - Construct: integrations:keyspaces:[8-char-hash]:efficy-enterprise-12-1
   - Store in discriminator mapping table
4. Contact keyspace creation:
   - Create keyspace in keyspace DB
   - Store keyspace ID alongside discriminator
5. Entity loop (Efficy 12.1 has only contact):
   - Entity type: 'contact' → discriminator already created, skip
6. Event listener registration:
   - For each active form campaign with 'sync to CRM' enabled:
     - Register listener on 'start viewed' event
     - Register listener on 'submit' event
7. Installation complete; Efficy 12.1 contact sync begins
```

### Procedure: Submit Form with Efficy 12.1 Sync Enabled
```
1. Form published with 'sync to CRM' option enabled (Efficy 12.1 integration installed)
2. User fills form, clicks submit
3. Submit event triggered in Apsis
4. Event listener fires; Form Submission Routing called with:
   - profileKey: <Apsis profile ID>
   - identifyingData: {email, phone, crmId (if known)}
   - fields: <form field values>
5. HTTP POST to Efficy 12.1 form submission endpoint:
   - Payload: {profileKey, identifyingData, fields}
6. Efficy 12.1 processes submission:
   - If email exists, returns existing contact ID
   - If new email, creates contact, returns new contact ID
7. Apsis receives response: {entityType: 'contact', recordId: <CRM ID>}
8. Keyspace lookup:
   - Query discriminator mapping: (sectionId, crmType='efficy-enterprise-12-1', entityType='contact')
   - Retrieve keyspaceId
9. Store CRM ID in contact keyspace:
   - Insert/update: profileKey → recordId in keyspaceId
10. Form submission complete; contact now linked in Apsis
```

### Procedure: Download Contacts from Efficy 12.1
```
1. Sync trigger: scheduled or on-demand
2. Efficy 12.1 sync endpoint called:
   - Query/webhook returns list of contacts with structure: {id, email, phone, ...}
3. For each contact returned:
   - entityType: 'contact' (Efficy 12.1 returns only contacts)
   - Keyspace lookup: query (sectionId, crmType='efficy-enterprise-12-1', entityType='contact') → keyspaceId
   - Update/insert contact in keyspaceId:
     - CRM ID → Apsis profileKey mapping
     - Contact fields sync to Apsis profile
4. Sync complete; Apsis profile data matches Efficy 12.1
```

### Procedure: Export Consent Changes to Efficy 12.1
```
1. Trigger: Apsis user changes contact consent state (email opt-in/out, SMS, etc.)
2. Consent change event captured
3. Consent Export Logic called:
   - Query contact keyspace: profileKey → recordId mappings
   - Filter: entityType='contact' (Efficy 12.1 has only contacts; no filtering needed)
   - Collect consent state for all updated contacts
4. HTTP POST to Efficy 12.1 consent endpoint:
   - Payload: [{recordId, consentState: {email: true/false, sms: true/false, ...}}, ...]
5. Efficy 12.1 updates contact consent state
6. Export complete; Efficy 12.1 consent matches Apsis
```

### Procedure: Uninstall Efficy 12.1 Integration
```
1. Uninstall trigger: Customer disables Efficy 12.1 integration
2. Event listener cleanup:
   - For each form campaign with listeners:
     - Deregister 'start viewed' event listener
     - Deregister 'submit' event listener
3. Form 'sync to CRM' option becomes hidden/disabled (if UI reloads)
4. Keyspace NOT deleted (intentional; preserves historical data)
   - Discriminator remains in mapping table
   - Contact keyspace data remains in keyspace DB
5. Uninstall complete; form submissions no longer sent to Efficy 12.1
```

### Procedure: Reinstall Efficy 12.1 Integration (Reuse Keyspace)
```
1. Reactivation trigger: Customer re-enables Efficy 12.1 integration
2. Installation Manager.main_install() called with same parameters as initial install
3. Discriminator lookup:
   - Query mapping table: (sectionId, crmType='efficy-enterprise-12-1', entityType='contact')
   - If exists: retrieve existing keyspaceId (reuse)
   - If not exists: create new keyspace
4. Event listener registration:
   - For each active form campaign with 'sync to CRM' enabled:
     - Register listeners (same as first install)
5. Historical data accessible:
   - All contact records and event history from previous installation now visible
   - Profiles re-linked via existing CRM ID mappings
6. Reinstall complete; re-engagement campaigns can resume with full history
```

### Procedure: GDPR Delete Contact from Efficy 12.1 (Manual)
```
1. GDPR deletion request received for contact
2. Delete in Apsis:
   - Query contact keyspace for profileKey or CRM ID
   - Hard-delete profile record from keyspace
   - Remove event history
3. Delete in Efficy 12.1 (if required by law):
   - HTTP DELETE to Efficy 12.1 contact endpoint: /contacts/{recordId}
   - Efficy 12.1 hard-deletes contact
4. Confirmation:
   - Log deletion in audit trail
   - Send confirmation to data owner
```

---

## 11. Deployment & Operations Guide

### Environment Configuration (To Be Documented)
```env
# Efficy Enterprise 12.1 Integration
EFFICY_12_1_ENDPOINT_URL=https://api.efficy.com/
EFFICY_12_1_API_KEY=<secure key>
EFFICY_12_1_FORM_SUBMIT_PATH=/integration/form-submit
EFFICY_12_1_CONTACT_DOWNLOAD_PATH=/integration/contacts
EFFICY_12_1_CONSENT_EXPORT_PATH=/integration/consent/update

# Keyspace Configuration
KEYSPACE_DISCRIMINATOR_FORMAT=integrations:keyspaces:{hash}:{crmType}
KEYSPACE_HASH_LENGTH=8

# Form Sync Configuration
FORM_SYNC_EVENT_TYPES=['start viewed', 'submit']
FORM_SYNC_LISTENER_CLEANUP_ON_UNINSTALL=true

# Error Handling
CRM_SUBMISSION_RETRY_ATTEMPTS=3
CRM_SUBMISSION_RETRY_BACKOFF_MS=1000
CRM_SUBMISSION_TIMEOUT_MS=30000
```

### Monitoring & Alerting (To Be Implemented)
```
Key metrics:
- Form submissions to Efficy 12.1 (success rate, latency)
- Contact downloads from Efficy 12.1 (sync frequency, record count)
- Consent export to Efficy 12.1 (success rate, export count)
- Keyspace lookup failures (rate, context)
- Event listener registration/deregistration (count, errors)

Alerts:
- Form submission failure rate > 5%
- Efficy 12.1 API response time > 5 seconds
- Keyspace lookup failure
- Contact download sync failure
```

---

**Generated**: 2026-03-25  
**Based on**: Keyspaces in Integrations part 2.md (1 KT session)  
**Subdomain**: efficy enterprise 12.1 within Apsis One Integrations  
**Confidence Level**: Medium-High (architecture well-documented; implementation details sparse)