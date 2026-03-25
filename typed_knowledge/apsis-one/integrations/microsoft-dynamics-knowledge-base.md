---
title: Apsis One Integrations — microsoft dynamics Knowledge Base
subdomain: microsoft dynamics
generated: 2026-03-25T12:40:46.853Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (1 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — microsoft dynamics Knowledge Base](#apsis-one-integrations-microsoft-dynamics-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Installation Manager](#installation-manager)
    - [Entity-Specific Keyspaces (Contact & Lead)](#entity-specific-keyspaces-contact-lead)
    - [Form Sync Event Listener System](#form-sync-event-listener-system)
    - [Form Submission Routing](#form-submission-routing)
    - [Profile Merging Engine](#profile-merging-engine)
    - [Consent Export Logic](#consent-export-logic)
    - [Keyspace Discriminator Generation & Lookup](#keyspace-discriminator-generation-lookup)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication](#authentication)
    - [Error Handling](#error-handling)
    - [Logging & Monitoring](#logging-monitoring)
    - [Deployment & Configuration Management](#deployment-configuration-management)
    - [GDPR & Data Retention](#gdpr-data-retention)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Master Data & Authority](#master-data-authority)
    - [Entity Separation & Keyspaces](#entity-separation-keyspaces)
    - [Form Sync & Routing](#form-sync-routing)
    - [Profile Merging Constraints](#profile-merging-constraints)
    - [Entity Type Handling](#entity-type-handling)
    - [Keyspace Lifecycle](#keyspace-lifecycle)
    - [Historical Data Preservation](#historical-data-preservation)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Issue #1: Microsoft Dynamics Tribe Variant Dynamic Entity Behavior](#issue-1-microsoft-dynamics-tribe-variant-dynamic-entity-behavior)
    - [Issue #2: Custom Keyspace Behavior Request (JoinCX Customer)](#issue-2-custom-keyspace-behavior-request-joincx-customer)
    - [Issue #3: Hash Collision Risk in Keyspace Discriminator](#issue-3-hash-collision-risk-in-keyspace-discriminator)
    - [Issue #4: Silent Keyspace Lookup Failures](#issue-4-silent-keyspace-lookup-failures)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence ✓](#high-confidence-)
    - [Medium Confidence ⚠️](#medium-confidence-)
    - [Low Confidence / Unknown ⚠️❓](#low-confidence-unknown-)
    - [Conflicts / Disagreements](#conflicts-disagreements)
    - [Gaps Requiring Clarification](#gaps-requiring-clarification)
  - [9. Tribal Knowledge](#9-tribal-knowledge)
    - [Design Philosophy](#design-philosophy)
    - [Common Gotchas](#common-gotchas)
    - [Optimization Opportunities](#optimization-opportunities)
    - [Non-Obvious Facts](#non-obvious-facts)
  - [10. References & Sources](#10-references-sources)
  - [Implementation Checklist for Microsoft Dynamics Integration](#implementation-checklist-for-microsoft-dynamics-integration)

---

# Apsis One Integrations — microsoft dynamics Knowledge Base

## 1. Subdomain Overview

The Microsoft Dynamics subdomain covers the integration between Apsis One and Microsoft Dynamics CRM systems, enabling bidirectional synchronization of contacts and leads, form submission routing, and consent management. This subdomain manages keyspace-based entity storage for contacts and leads, handles form submissions with CRM sync capabilities, coordinates profile merging between entity types, and ensures the CRM system remains the authoritative source of contact data. Boundaries: covers form sync configuration, entity keyspace creation/management, contact/lead downloads, and consent exports to Microsoft Dynamics; does NOT cover custom entity types beyond contacts/leads or non-standard authentication mechanisms.

---

## 2. Architecture Map

```
Microsoft Dynamics Integration Flow
├── Installation Phase
│   ├── Installation Manager (triggered on CRM integration install)
│   ├── Main Keyspace Creation (contact entity)
│   │   └── Discriminator: integrations:keyspaces:[8-char-hash]:microsoft-dynamics
│   ├── Lead Keyspace Creation (if supported by variant)
│   │   └── Discriminator: integrations:keyspaces:[8-char-hash]:microsoft-dynamics:lead
│   └── Event Listener Registration (for form campaigns with sync enabled)
│
├── Form Submission Flow
│   ├── Form Published with "sync to CRM" enabled
│   ├── Event Listeners Registered ('start viewed', 'submit' events)
│   ├── User Submits Form
│   │   └── Payload: {profileKey, identifyingData: {email, phone, crmId, leadId}, fields: {...}}
│   ├── Form Submission Routing → Microsoft Dynamics
│   ├── CRM Response: {entityType: 'contact' or 'lead', recordId}
│   ├── Keyspace Lookup by Entity Type
│   ├── ID Storage in Correct Entity Keyspace
│   └── Profile Merge (if lead converted to contact)
│
├── Data Download Flow
│   ├── CRM Sync Triggered (scheduled or event-based)
│   ├── Microsoft Dynamics Returns: {entityType, id, data}
│   ├── Discriminator Lookup by Entity Type
│   ├── Keyspace Retrieval
│   └── Entity Update in Correct Keyspace
│
├── Consent Export Flow
│   ├── Consent Changes in Apsis
│   ├── Export to Microsoft Dynamics (contacts only)
│   └── **CRITICAL**: Leads/entities NOT included in export
│
└── Uninstallation Phase
    └── Event Listeners Removed (keyspace preserved for re-engagement)
```

**Data Flow Detail:**
- CRM ID ↔ Keyspace Discriminator (hash + logical name) → Keyspace DB lookup → Contact/Lead profile updates
- Entity type from CRM determines which keyspace receives ID and which events are processed
- Merges occur within integration keyspaces only (contact ↔ lead); external merges prohibited

---

## 3. Module Reference

### Installation Manager
- **Purpose**: Bootstraps keyspaces and event listeners when Microsoft Dynamics integration is installed on an account section.
- **Key files**: `Installation Manager (main install function)` [path not specified in transcript]
- **Tech stack**: Apsis One Integrations platform (language/framework not specified)
- **Data flow**: 
  - **In**: Integration trigger + CRM logical name + account section ID
  - **Out**: Keyspace discriminators stored in database; event listeners registered for forms
- **Business rules**: 
  - Creates main keyspace with discriminator: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics`
  - Loops through entity types supported by Microsoft Dynamics variant and creates separate keyspaces for each
  - On reinstallation: **reuses existing keyspace** (does NOT delete on uninstall, does NOT create new on reinstall)
  - Event listeners initialized only for forms with 'sync to CRM' enabled
- **Configuration**: 
  - Entity types per Microsoft Dynamics variant: `{contact}` or `{contact, lead}` depending on variant
  - Hash length: 8 characters (first 8 chars of account section discriminator)
- **Integration points**: 
  - Microsoft Dynamics CRM (entity type metadata queried during install—method not specified)
- **Gotchas**: 
  - ⚠️ Reinstalling reuses old keyspace; historical data is NOT lost
  - ⚠️ Uninstalling does NOT delete keyspace; data remains for re-engagement scenarios
  - Entity types must match what specific Microsoft Dynamics variant supports (contact always; lead varies)
- **Tribal knowledge**: 
  - The decision to preserve keyspaces on uninstall is deliberate for "win-back" customer scenarios
  - 8-character hash provides uniqueness across multiple installations; no reverse resolution needed

---

### Entity-Specific Keyspaces (Contact & Lead)
- **Purpose**: Separate storage containers for contact and lead entities to prevent data model conflicts and incorrect consent exports.
- **Key files**: `Installation Manager (entity for loop)` [path not specified]
- **Tech stack**: Apsis One Integrations platform keyspace database
- **Data flow**: 
  - **In**: Entity type from Microsoft Dynamics (contact | lead) + record ID + identifying data
  - **Out**: Keyspace ID for downstream operations (store ID, query records, updates)
- **Business rules**: 
  - Contact keyspace receives contact IDs; lead keyspace receives lead IDs
  - Contact keyspace is **the only keyspace** that participates in consent exports back to CRM
  - Lead IDs must NEVER be stored in contact keyspace (would cause incorrect consent exports)
  - Contacts and leads remain separate until explicit merge request from CRM
  - Keyspace format: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics` (contact) and `integrations:keyspaces:[8-char-hash]:microsoft-dynamics:lead` (lead)
- **Configuration**: 
  - Discriminator suffixes: none for contact (main), `:lead` for lead entity keyspace
  - Stored in database during installation for later lookup
- **Integration points**: 
  - Form Submission Routing (determines which keyspace receives ID)
  - Profile Merging Engine (merges lead with contact upon conversion)
  - Consent Export Logic (contacts only)
  - CRM data downloads (entity type determines keyspace target)
- **Gotchas**: 
  - ⚠️ If Microsoft Dynamics variant doesn't support leads, lead keyspace is never created
  - ⚠️ Lead keyspace may exist but be unused if CRM never returns lead responses (e.g., all submissions convert to contacts immediately)
- **Tribal knowledge**: 
  - Separate keyspaces prevent silhouette-like issues seen in E-deal where wrong entity type in export causes data sync problems

---

### Form Sync Event Listener System
- **Purpose**: Registers event listeners ('start viewed', 'submit') on form campaigns when 'sync to CRM' option is enabled; triggers CRM synchronization on form submission.
- **Key files**: None specified; likely in `Form Campaign Configuration` module [path not specified]
- **Tech stack**: Apsis One Integrations platform event listener framework
- **Data flow**: 
  - **In**: Form published with 'sync to CRM' enabled
  - **Out**: Event listeners registered for form; removed on integration uninstall
- **Business rules**: 
  - Event listeners ONLY registered if:
    - Integration is installed AND
    - 'sync to CRM' option is explicitly enabled on form
  - 'sync to CRM' option is **hidden/unavailable** if integration not installed
  - Listeners trigger on 'start viewed' and 'submit' events
  - Listeners removed entirely on integration uninstall (but form configuration preserved)
- **Configuration**: 
  - Event listener registration triggered by: Form published with 'sync to CRM' enabled
  - No explicit configuration needed; automatic on form publish
- **Integration points**: 
  - Form Campaign Configuration UI (where 'sync to CRM' toggle is defined)
  - Form Submission Handler (receives submit event from listener)
  - Installation Manager (removes listeners on uninstall)
- **Gotchas**: 
  - ⚠️ 'sync to CRM' option not visible in UI unless integration installed; customers may not realize it's unavailable
  - ⚠️ If integration is uninstalled while forms have sync enabled, listeners are removed but form configuration retains 'sync to CRM' flag (becomes inert)
  - ⚠️ Under normal circumstances, form submissions will NOT reach uninstalled/inactive integrations (safety feature)
- **Tribal knowledge**: 
  - This is an intentional safety mechanism to prevent orphaned form submissions to inactive integrations

---

### Form Submission Routing
- **Purpose**: Routes form submissions to Microsoft Dynamics CRM, handles CRM responses, stores IDs in correct entity keyspaces, and triggers profile merges.
- **Key files**: None specified [path not specified]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**: 
  - **In**: Form submission with profileKey + identifyingData (email, phone, crmId, leadId) + fields
  - **CRM**: `{entityType: 'contact' | 'lead', recordId}` response
  - **Out**: Record ID stored in correct keyspace; merge triggered if needed
- **Business rules**: 
  - Form submission payload structure: `{profileKey, identifyingData: {email, phone, crmId, leadId}, fields: {...}}`
  - Profile key added by Apsis to identify profile on Apsis side
  - CRM response includes entity type (contact | lead) and new or existing record ID
  - Entity type in response determines which keyspace receives the ID
  - If response indicates lead → store ID in lead keyspace + merge with contact keyspace (if applicable)
  - If response indicates contact → store ID in contact keyspace
  - Identifying data (email, phone, existing IDs) used by CRM to determine if contact exists or new contact created
- **Configuration**: 
  - Form must have 'sync to CRM' enabled and integration must be installed
  - Form must contain email, phone, or other identifying fields for CRM lookup
- **Integration points**: 
  - Microsoft Dynamics CRM (REST protocol, bidirectional form submission)
  - Entity-Specific Keyspaces (stores returned IDs)
  - Profile Merging Engine (triggered on lead response)
- **Gotchas**: 
  - ⚠️ If integration not installed, 'sync to CRM' option hidden and form never routes to CRM
  - ⚠️ Entity type in CRM response MUST match an expected type (contact | lead); unexpected type could cause routing failure
  - ⚠️ Identifying data format must match what CRM expects; mismatched format may cause CRM to create duplicate records
  - ⚠️ Form fields payload structure must match CRM API expectations (detail not specified in transcript)
- **Tribal knowledge**: 
  - The profileKey is critical to tie Apsis profile to CRM record; it's added by Apsis, not provided by customer

---

### Profile Merging Engine
- **Purpose**: Merges lead/contact records when CRM indicates conversion or form response; operates strictly within integration entity keyspaces.
- **Key files**: None specified [path not specified]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**: 
  - **In**: Lead ID from CRM response + Profile key from form submission + Contact keyspace data
  - **Out**: Merged record preventing duplicate creation on re-submission
- **Business rules**: 
  - Merge triggered when:
    - CRM indicates lead → contact conversion OR
    - Form submission receives multiple entity type indicators
  - Merge operation: combines lead ID from lead keyspace with profile key from contact/export keyspace
  - Merges ONLY occur within integration entity keyspaces (lead ↔ contact within single CRM integration)
  - **PROHIBITED**: Merging CRM-originated contacts with other keyspace profiles (e.g., email keyspace, Apsis-native profiles)
  - Merges initiated from CRM (via API or response) are acceptable; Apsis-initiated merges are not
  - CRM is master of merge decisions; Apsis should never initiate merges
- **Configuration**: No explicit configuration; automatic when merge condition detected
- **Integration points**: 
  - Form Submission Routing (receives lead response triggering merge)
  - Entity-Specific Keyspaces (accesses both lead and contact keyspaces)
  - CRM system (provides merge indication via API or response)
- **Gotchas**: 
  - ⚠️ Merging CRM contact with email keyspace profile in Apsis will NOT propagate back to CRM; creates data inconsistency
  - ⚠️ If merge request never arrives from CRM (e.g., silhouette stuck in separate keyspace), record remains unmerged indefinitely
  - ⚠️ Merging should happen in CRM first, then propagate to Apsis; reverse order causes data integrity issues
- **Tribal knowledge**: 
  - The silhouette-like scenario is why separate keyspaces exist—to hold incomplete records until CRM confirms merge

---

### Consent Export Logic
- **Purpose**: Sends contact consent changes back to Microsoft Dynamics; critically safeguards against exporting non-contact entities.
- **Key files**: None specified [path not specified]
- **Tech stack**: Apsis One Integrations platform consent export module
- **Data flow**: 
  - **In**: Consent changes in Apsis for contacts
  - **Out**: Consent payload to Microsoft Dynamics (contacts only)
- **Business rules**: 
  - **CRITICAL**: Only contacts are exported; leads are NEVER included in consent exports
  - Export payload constructed from contact keyspace only
  - If lead ID mistakenly stored in contact keyspace, consent export would include lead (incorrect behavior)
  - Export sent only to Microsoft Dynamics for records that originated from CRM
- **Configuration**: Automatic; triggered by consent changes
- **Integration points**: 
  - Contact Keyspace (data source for export)
  - Microsoft Dynamics CRM (REST endpoint for consent export)
- **Gotchas**: 
  - ⚠️ If lead ID stored in contact keyspace by mistake, export payload will include lead record ID (causes CRM to fail or create inconsistency)
  - ⚠️ Export only processes IDs from contact keyspace; IDs in lead keyspace are ignored (by design)
- **Tribal knowledge**: 
  - This is why separate keyspaces are so critical; a single keyspace would cause leads to be incorrectly included in consent exports

---

### Keyspace Discriminator Generation & Lookup
- **Purpose**: Generates unique discriminators for keyspaces and enables database lookups to retrieve keyspace IDs for operations.
- **Key files**: `Installation Manager (discriminator generation logic)` [path not specified]
- **Tech stack**: Apsis One Integrations platform
- **Data flow**: 
  - **Generation**: Account section discriminator → 8-char hash extraction → combined with CRM logical name → discriminator string
  - **Lookup**: Entity type + CRM type → discriminator construction → database query → keyspace ID returned
- **Business rules**: 
  - Discriminator format: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics` (contact) or `integrations:keyspaces:[8-char-hash]:microsoft-dynamics:lead` (lead)
  - 8-character hash derived from account section discriminator (first 8 characters)
  - Hash provides uniqueness across multiple installations of same CRM on same account section
  - Discriminators stored in database during installation
  - No reverse resolution of hash needed; hash is one-way (intentional)
  - Lookup performed during form submission responses, CRM downloads, and consent exports
- **Configuration**: 
  - Hash length: 8 characters (hardcoded; rationale for this specific length not documented)
  - CRM logical name: `microsoft-dynamics` (constant for all Microsoft Dynamics integrations)
- **Integration points**: 
  - Installation Manager (creates and stores discriminators)
  - Form Submission Routing (looks up keyspace by entity type)
  - CRM Data Download (looks up keyspace by entity type)
  - Consent Export Logic (looks up contact keyspace)
- **Gotchas**: 
  - ⚠️ 8-character hash collision theoretically possible across different sections (low probability; mitigation not documented)
  - ⚠️ Hash is case-sensitive (case sensitivity not explicitly stated but assumed)
  - ⚠️ If discriminator not found in database, lookup fails and operation may fail silently
  - ⚠️ Hash cannot be used to reverse-identify source section; it's one-way by design
- **Tribal knowledge**: 
  - The intentional one-way design means you can't recover which section a keyspace came from; this is acceptable because discriminators are stored in the database

---

## 4. Cross-Cutting Concerns

### Authentication
- **Status**: ⚠️ **NOT DOCUMENTED** in transcript
- **Scope**: How Apsis authenticates to Microsoft Dynamics (OAuth, API key, service account, etc.)
- **Impact**: Form submissions, data downloads, consent exports all require auth to CRM
- **Recommendation**: Clarify auth mechanism for each Microsoft Dynamics variant (Legacy, Siteshop)

### Error Handling
- **Status**: ⚠️ **NOT DOCUMENTED** in transcript
- **Known scenarios needing error handling**:
  - CRM returns unexpected entity type (not contact | lead)
  - Discriminator lookup fails in database
  - Form submission payload malformed or missing identifying data
  - Microsoft Dynamics API returns error on form submission
  - Keyspace lookup returns no results
- **Recommendation**: Document retry logic, failure modes, and fallback behavior

### Logging & Monitoring
- **Status**: ⚠️ **NOT DOCUMENTED** in transcript
- **Critical events to log**:
  - Form submission routing to CRM
  - Entity ID storage in keyspace (successful/failed)
  - Profile merge operations
  - Consent export to CRM
  - Keyspace lookup failures
  - Installation/uninstallation of Microsoft Dynamics integration
- **Recommendation**: Define log levels and alerting thresholds for integration failures

### Deployment & Configuration Management
- **Status**: ⚠️ **PARTIALLY DOCUMENTED**
- **Env vars/config**: Entity types per Microsoft Dynamics variant should be configurable
- **Database**: Keyspace discriminator mapping table required; schema not specified
- **Variants**: Support for "Legacy" and "Siteshop" Microsoft Dynamics variants mentioned but not defined
- **Recommendation**: Document variant-specific configurations and deployment checklist

### GDPR & Data Retention
- **Status**: **PARTIALLY DOCUMENTED**
- **Principle**: Profiles synced to Apsis from CRM remain in Apsis unless explicitly removed
- **Exception**: GDPR deletion requests trigger profile removal
- **Keyspace behavior**: Keyspaces are preserved on uninstall to support re-engagement; GDPR cleanup must explicitly remove records
- **Recommendation**: Document GDPR deletion procedure and its interaction with keyspace preservation

---

## 5. Business Rules Reference

### Master Data & Authority
1. **CRM is always the master of contact data**
   - CRM-originated profiles are authoritative
   - Apsis should not override CRM data
   - Changes to CRM records should propagate to Apsis
   - Reverse changes (Apsis → CRM) limited to consent and form submissions
   - **Context**: Data synchronization, profile merging, consent management
   - **Exception**: None documented

2. **Profiles synced to Apsis remain unless explicitly removed**
   - Historical event data preserved for re-engagement campaigns
   - Uninstalling integration does NOT delete keyspace
   - Reinstalling reuses existing keyspace and all historical data
   - **Context**: Keyspace lifecycle, integration uninstallation
   - **Exception**: GDPR deletion requests explicitly remove profiles

### Entity Separation & Keyspaces
3. **Contacts and leads remain separate until explicit merge from CRM**
   - Contact keyspace and lead keyspace are isolated
   - IDs stored in correct keyspace based on CRM response entity type
   - Separate keyspaces prevent consent export errors
   - **Context**: Entity-specific keyspace creation, form submission routing
   - **Exception**: None; separation is mandatory

4. **Lead IDs must NEVER be stored in contact keyspace**
   - Lead ID in contact keyspace would cause lead inclusion in consent exports
   - Incorrect consent export creates data inconsistency in CRM
   - **Context**: Form submission ID storage, entity validation
   - **Exception**: None; violating this rule breaks consent sync

5. **Consent exports include ONLY contacts, never leads**
   - Contact keyspace is source of consent export
   - Lead keyspace IDs excluded from export payloads
   - Protects data integrity in CRM
   - **Context**: Consent export logic, keyspace usage
   - **Exception**: None documented

### Form Sync & Routing
6. **Form sync requires 'sync to CRM' option explicitly enabled**
   - 'sync to CRM' toggle must be set on form campaign
   - Option only visible/available if integration installed
   - Under normal circumstances, uninstalled integration forms do NOT sync
   - **Context**: Form campaign configuration, event listener registration
   - **Exception**: None under normal circumstances

7. **Form submission payload must include identifying data**
   - Email, phone, or existing CRM ID required for CRM to lookup contact
   - Profile key added by Apsis to correlate Apsis profile with form submission
   - Fields property contains customer-provided form data
   - **Context**: Form submission routing to CRM
   - **Exception**: None; form cannot route to CRM without identifying data

### Profile Merging Constraints
8. **Merging CRM-originated contacts with non-CRM keyspaces is prohibited**
   - Do NOT merge CRM contact (from contact keyspace) with email keyspace profile
   - Do NOT merge CRM contact with Apsis-native profile
   - Merges within integration entity keyspaces only acceptable (lead ↔ contact within same CRM)
   - Merges must originate from CRM, not Apsis
   - **Context**: Profile management, data consistency
   - **Rationale**: CRM doesn't know about Apsis merges; causes CRM/Apsis divergence
   - **Exception**: Merges initiated by CRM system itself are acceptable

### Entity Type Handling
9. **Entity type from CRM response determines keyspace destination**
   - Contact type → contact keyspace receives ID
   - Lead type → lead keyspace receives ID
   - CRM response entity type is authoritative
   - Unexpected entity types may cause routing failure
   - **Context**: Form submission routing, data download processing
   - **Exception**: None; entity type must be valid

10. **Entity keyspaces must match CRM capabilities**
    - Microsoft Dynamics variants may support only contacts OR contacts + leads
    - Lead keyspace created only if CRM variant supports leads
    - No keyspaces created for unsupported entity types
    - **Context**: Installation Manager, entity-specific keyspace creation
    - **Exception**: None; limited by CRM capabilities

### Keyspace Lifecycle
11. **Keyspaces are reused on reinstallation, never recreated**
    - Uninstalling does NOT delete keyspace
    - Reinstalling same integration reuses old keyspace and all data
    - Enables win-back scenarios with preserved event history
    - **Context**: Installation/uninstallation procedures
    - **Exception**: None; reuse is mandatory design

12. **Keyspace discriminators are immutable once created**
    - Discriminator format: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics[:lead]`
    - Hash derived from account section (fixed per section)
    - CRM logical name is constant
    - Discriminators stored in database for lookups
    - **Context**: Discriminator generation, database schema
    - **Exception**: None documented

### Historical Data Preservation
13. **Historical event data remains in keyspace even after uninstallation**
    - Event history tied to keyspace, not to integration status
    - Uninstallation does NOT clear event history
    - Historical data enables re-engagement campaigns if customer reactivates integration
    - **Context**: Keyspace preservation rationale, business continuity
    - **Exception**: GDPR deletion requests explicitly remove data

---

## 6. Known Issues & Workarounds

### Issue #1: Microsoft Dynamics Tribe Variant Dynamic Entity Behavior
- **Severity**: Medium
- **Description**: Tribe's UI includes a dynamic entity dropdown allowing customers to select contact, lead, or custom entity type. Apsis doesn't know in advance which entity will be selected. Apsis currently bootstraps a lead keyspace to handle flexibility, but Tribe never actually responds with 'lead' type—they respond with whatever the dropdown selected. This creates confusion about whether the lead keyspace is ever used.
- **Affected component**: Form Submission Routing, Entity Keyspaces, Tribe Integration (Microsoft Dynamics variant)
- **Workaround**: Current workaround is to treat 'lead' as a virtual entity representing Tribe's dynamic dropdown capability. Long-term simplification attempted by Erik (team member), but no timeline committed.
- **Root cause**: Tribe's API design doesn't align with standard lead/contact entity model used by other CRMs.
- **Impact**: Lead keyspace may be created during installation but never receive data if customer's dropdown always selects contact or custom entity.

### Issue #2: Custom Keyspace Behavior Request (JoinCX Customer)
- **Severity**: Low
- **Description**: Custom integration customer (JoinCX) requested use of email keyspace instead of CRM keyspace for their installation. However, implementing this in the general connector code would apply to ALL JoinCX customers globally, which is undesirable if behavior is customer-specific.
- **Affected component**: Installation Manager, Keyspace System, JoinCX Integration
- **Workaround**: Under investigation; Wednesday follow-up scheduled between Erik and Lukasz to explore non-code solutions (per-customer configuration, environment variables, or database-driven rules).
- **Root cause**: System not designed for per-customer keyspace customization; installation logic is global, not per-account.
- **Impact**: If implemented as code, would apply to ALL JoinCX customers; if not implemented, single customer has unsupported use case.

### Issue #3: Hash Collision Risk in Keyspace Discriminator
- **Severity**: Low (unlikely to occur)
- **Description**: 8-character hash component of discriminator provides uniqueness across installations, but theoretical collision risk exists across different sections. No collision detection or mitigation mechanism documented.
- **Affected component**: Keyspace Discriminator Generation
- **Workaround**: Collision is extremely unlikely given 8 characters (62^8 ≈ 218 trillion combinations); no active mitigation required unless collisions observed.
- **Root cause**: Hash-based uniqueness is by design; reverse resolution not implemented, so no way to disambiguate collisions.
- **Impact**: Negligible in practice; monitoring could be added if collisions detected.

### Issue #4: Silent Keyspace Lookup Failures
- **Severity**: Medium
- **Description**: If keyspace discriminator not found in database during lookup, operation may fail silently without clear error indication. No error handling documented for missing discriminators.
- **Affected component**: Form Submission Routing, CRM Data Download, Consent Export Logic
- **Workaround**: Unknown; no documented workaround. Recommend adding logging and error handling for lookup failures.
- **Root cause**: Database query returns empty result; unclear how system handles missing keyspace.
- **Impact**: Form submissions may silently fail to route to CRM; data downloads may not reach correct keyspaces; consent exports may not process.

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Keyspace** | Unique identifier and storage container for a specific entity type (contact, lead) within a specific CRM installation (Microsoft Dynamics) on a specific account section. Identified by discriminator. |
| **Discriminator** | Unique string identifier for a keyspace in format: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name][:entity-type]`. Used for database lookup to retrieve keyspace ID. Example: `integrations:keyspaces:a1b2c3d4:microsoft-dynamics` (contact) or `integrations:keyspaces:a1b2c3d4:microsoft-dynamics:lead` (lead). |
| **8-char Hash** | First 8 characters of account section discriminator, used in keyspace discriminator to ensure uniqueness across multiple installations of same CRM on same account section. |
| **Contact** | Primary entity type in Microsoft Dynamics representing a customer or prospect contact record. Stored in contact keyspace. Participates in consent exports. |
| **Lead** | Secondary entity type in Microsoft Dynamics (if variant supports) representing a potential customer record. May be converted to contact. Stored in lead keyspace. NEVER included in consent exports. |
| **Entity Keyspace** | Keyspace dedicated to specific entity type (lead) beyond main contact keyspace. Created during installation only if CRM variant supports that entity type. |
| **Main Keyspace** | Primary keyspace for contact entity type, created first during installation. Also called "contact keyspace." |
| **Form Sync** | Feature enabling form submissions to be automatically sent to Microsoft Dynamics CRM. Enabled via explicit 'sync to CRM' toggle on form campaign. Only visible/available if integration installed. |
| **sync to CRM** | Toggle option on form campaign that enables form sync. Must be explicitly enabled per form. Option hidden if integration not installed. |
| **Event Listener** | Mechanism that registers for specific events ('start viewed', 'submit') on form campaigns when sync to CRM enabled. Triggers CRM synchronization on form submission. Removed on integration uninstall. |
| **Profile Key** | Unique identifier for an Apsis profile. Added to form submission payloads by Apsis to identify profile on Apsis side. Tied to profile record in Apsis One. |
| **CRM ID / Record ID** | Unique identifier for a contact or lead in Microsoft Dynamics. Returned in CRM response to form submissions and data downloads. Stored in corresponding entity keyspace. |
| **Entity Type** | Classification of record in Microsoft Dynamics (contact, lead, etc.). Returned in CRM response to form submission. Determines which keyspace receives record ID. |
| **Merge Request** | Request from Microsoft Dynamics to merge two related records (e.g., lead → contact conversion). Triggers merge between lead keyspace and contact keyspace within integration. |
| **Consent Export** | Process of sending contact consent changes from Apsis back to Microsoft Dynamics. Includes ONLY contacts from contact keyspace; leads NEVER exported. |
| **Master Data / CRM is Master** | Design principle: Microsoft Dynamics system is authoritative source of contact data. Apsis should not override CRM data; merges should originate from CRM. |
| **Silhouette** | (Related concept from E-deal) Temporary or incomplete contact record. Not directly applicable to Microsoft Dynamics but similar concept applies to leads (separate entity type until merged with contact). |
| **Export Keyspace** | (Related concept) Keyspace containing profile key, used in merges to tie together entity ID and profile key. Specific terminology may vary in Microsoft Dynamics context. |
| **Installation** | Process of activating Microsoft Dynamics integration on an account section. Creates keyspaces and registers event listeners. |
| **Uninstallation** | Process of deactivating Microsoft Dynamics integration on an account section. Removes event listeners but preserves keyspaces and data. |
| **Reinstallation** | Process of reactivating Microsoft Dynamics integration after uninstallation. Reuses existing keyspace and all historical data. |
| **Variant** | Version or flavor of Microsoft Dynamics supported (e.g., Legacy, Siteshop). Different variants may support different entity types. |
| **Identifying Data** | Information used by CRM to lookup/create contact records: email, phone, existing CRM ID, existing lead ID. Required in form submission payload. |

---

## 8. Confidence Notes

### High Confidence ✓
- Keyspace discriminator format and uniqueness strategy
- Never delete keyspaces on uninstallation; reuse on reinstallation
- Entity-specific keyspaces (contact, lead) separation
- Form sync requires explicit 'sync to CRM' option
- Consent exports include ONLY contacts, never leads
- Profile merging only within integration entity keyspaces
- CRM is master of contact data
- Form submission payload structure (profileKey, identifyingData, fields)

### Medium Confidence ⚠️
- Exact database schema for keyspace discriminator storage (not documented in detail)
- Error handling for missing discriminators or lookup failures
- Silent failure modes for form submission routing
- Specific authentication mechanism for Microsoft Dynamics API
- Exact format of form fields payload sent to CRM
- Behavior of lead keyspace when lead entity not actually used by CRM variant

### Low Confidence / Unknown ⚠️❓
- Why specifically 8 characters for hash (not documented; possibly arbitrary)
- Hash collision detection/mitigation
- Reverse resolution mechanism (if any) for hash in discriminator
- Full structure of keyspace database table (only discriminator mapping implied)
- How entity types are queried from Microsoft Dynamics during installation
- Per-customer keyspace customization mechanisms
- Specific configuration for Microsoft Dynamics "Siteshop" variant
- Logging and monitoring strategy
- Retry logic and error recovery procedures
- Consent export payload structure and validation
- Archive/cleanup strategy for orphaned lead records

### Conflicts / Disagreements
- None documented in transcript. All architectural decisions appear aligned and consistently explained.

### Gaps Requiring Clarification
1. **Authentication**: How does Apsis authenticate to Microsoft Dynamics? OAuth 2.0? API key? Service account?
2. **Error handling**: What happens if form submission to CRM fails? Retry logic? Dead letter queue? Manual intervention?
3. **Database schema**: Exact structure of keyspace discriminator lookup table?
4. **Entity type discovery**: How are entity types determined at installation time? Hardcoded per variant or queried from CRM?
5. **Logging**: What events are logged? Which are errors vs. warnings vs. info?
6. **Variants**: Complete definition of "Legacy" and "Siteshop" Microsoft Dynamics variants and their entity type support?
7. **Tribe dynamic entity**: Current implementation status and timeline for simplification?
8. **JoinCX custom keyspace**: Status of investigation and planned solution for per-customer configuration?

---

## 9. Tribal Knowledge

### Design Philosophy
- **Keyspace preservation is intentional**: Uninstalling an integration does NOT delete the keyspace. This is deliberate to preserve historical event data and enable "win-back" customer scenarios where you want to resume engagement with historical profile context.
- **8-character hash is one-way by design**: You don't need to (and shouldn't try to) reverse-resolve the hash in a keyspace discriminator to identify the source section. The hash is for uniqueness only; discriminators are stored in the database for lookup.
- **Separate keyspaces prevent silent data sync errors**: Storing a lead ID in the contact keyspace would cause leads to be incorrectly included in consent exports back to Microsoft Dynamics. This silent data corruption is what the separate keyspace design prevents.

### Common Gotchas
- **E-deal silhouettes vs. Microsoft Dynamics leads**: While Microsoft Dynamics doesn't have "silhouettes," the concept is similar—separate entity keyspace prevents incorrect data export. Don't confuse terminology across CRM systems.
- **The 'sync to CRM' option disappears if integration not installed**: Customers may not realize this is why the option is grayed out. UI behavior should be explicit about what's required.
- **Tribe's dynamic entity dropdown complexity**: Tribe's lead is not a true lead entity type—it's a virtual representation of a customer-selected dropdown. This makes Tribe more annoying to work with than standard CRM systems.
- **Form sync under normal circumstances will never reach an uninstalled integration**: This is a safety feature. If you uninstall an integration, form submissions automatically stop routing to that CRM, even if 'sync to CRM' is still enabled on the form configuration.

### Optimization Opportunities
- **Silhouette/lead hold pattern**: Separate keyspace acts as a holding area for incomplete records (especially leads). This prevents duplicate profile creation if the same person resubmits a form.
- **Keyspace discriminator uniqueness without reverse resolution**: Using a hash instead of a reversible encoding saves database lookups and reverse-resolution logic. The tradeoff is that you can't identify the source section from the discriminator alone, but this is acceptable because discriminators are stored in the database.

### Non-Obvious Facts
- **Keyspace doesn't know about integration status**: The keyspace remains functional and queryable even after the integration is uninstalled. It's the event listeners that are removed; the keyspace itself persists. This is why data remains accessible for re-engagement campaigns.
- **Profile key ties Apsis profile to CRM record**: The profile key is added by Apsis to the form submission payload, not provided by the customer. It's critical for correlating Apsis-side profile records with CRM records.
- **CRM entity type response is authoritative**: The entity type returned by Microsoft Dynamics in the form submission response determines which keyspace receives the ID. If the CRM says it's a lead, it goes to the lead keyspace, regardless of what the form configuration expected.
- **Consent exports only use contact keyspace**: Even if you have lead IDs stored in the lead keyspace, consent exports will NEVER include them. This is enforced at the export logic level, not at the keyspace level. Separate keyspaces just make this easier to implement correctly.

---

## 10. References & Sources

- **Primary source**: Transcript from "Keyspaces in Integrations part 2.md" KT session (2026-03-25)
- **Discussed by**: Domain experts covering keyspace system, Installation Manager, Form Sync, Entity-Specific Keyspaces, Profile Merging, CRM ID Management, Form Submission Handling
- **Architecture decisions**: All high-confidence decisions documented with rationale and alternatives considered
- **Procedures**: 6 key procedures documented with prerequisites, frequency, and gotchas
- **Business rules**: 13 business rules categorized by domain area with context and exceptions

---

## Implementation Checklist for Microsoft Dynamics Integration

Use this checklist when implementing or troubleshooting Microsoft Dynamics integration:

- [ ] Verify keyspace discriminators stored in database after installation
- [ ] Confirm contact keyspace created with format: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics`
- [ ] Confirm lead keyspace created (if variant supports) with format: `integrations:keyspaces:[8-char-hash]:microsoft-dynamics:lead`
- [ ] Test form submission with 'sync to CRM' enabled; verify event listeners registered
- [ ] Test form submission identifies correct keyspace and stores ID in correct entity bucket
- [ ] Test lead-to-contact merge (if applicable); verify merge doesn't corrupt consent data
- [ ] Test consent export; verify ONLY contacts included, never leads
- [ ] Test uninstallation; verify keyspaces NOT deleted, event listeners removed
- [ ] Test reinstallation; verify keyspaces reused, historical data preserved
- [ ] Test form 'sync to CRM' option visibility: hidden if integration not installed
- [ ] Verify entity type from CRM response matches expected types (contact, lead)
- [ ] Test profile key propagation through form submission to CRM and back
- [ ] Validate discriminator lookup performance; ensure no N+1 queries
- [ ] Monitor keyspace lookup failures; log to error tracking system
- [ ] Document variant-specific entity type support (what each Microsoft Dynamics variant supports)
- [ ] Verify GDPR deletion removes profiles AND records from keyspaces
- [ ] Test merge prevention: CRM contact + email keyspace profile should NOT merge in Apsis

---

**Document Version**: 1.0  
**Last Updated**: 2026-03-25  
**Knowledge Base Confidence**: ~70% (good coverage of keyspace system and architecture decisions; gaps in auth, error handling, and logging)