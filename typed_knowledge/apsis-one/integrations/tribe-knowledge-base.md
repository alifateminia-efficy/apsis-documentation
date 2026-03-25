---
title: Apsis One Integrations — tribe Knowledge Base
subdomain: tribe
generated: 2026-03-25T12:40:46.854Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (1 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — tribe Knowledge Base](#apsis-one-integrations-tribe-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Installation Manager](#installation-manager)
    - [Keyspace System](#keyspace-system)
    - [Form Sync / Event Listener System](#form-sync-event-listener-system)
    - [Form Submission Routing](#form-submission-routing)
    - [Entity-Specific Keyspaces](#entity-specific-keyspaces)
    - [Profile Merging Engine](#profile-merging-engine)
    - [CRM ID Management](#crm-id-management)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication](#authentication)
    - [Error Handling](#error-handling)
    - [Logging](#logging)
    - [Deployment](#deployment)
    - [Consent Exports](#consent-exports)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Entity Separation & Data Isolation](#entity-separation-data-isolation)
    - [CRM as Master](#crm-as-master)
    - [Data Persistence](#data-persistence)
    - [Form Sync Features](#form-sync-features)
    - [Consent & Export](#consent-export)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Issue 1: Tribe Dynamic Entity Dropdown Behavior](#issue-1-tribe-dynamic-entity-dropdown-behavior)
    - [Issue 2: Custom Keyspace Configuration (JoinCX precedent, applies to Tribe)](#issue-2-custom-keyspace-configuration-joincx-precedent-applies-to-tribe)
    - [Issue 3: No Reverse Resolution of Keyspace Discriminator Hash](#issue-3-no-reverse-resolution-of-keyspace-discriminator-hash)
    - [Issue 4: Orphaned Silhouette/Lead Records](#issue-4-orphaned-silhouettelead-records)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence ✓](#high-confidence-)
    - [Medium Confidence ⚠️](#medium-confidence-)
    - [Low Confidence / Unresolved ❌](#low-confidence-unresolved-)
    - [Conflicts/Disagreements](#conflictsdisagreements)
    - [Gaps](#gaps)
  - [9. Tribal Knowledge (Non-Obvious Domain Facts)](#9-tribal-knowledge-non-obvious-domain-facts)
  - [10. Integration Points Detail](#10-integration-points-detail)
    - [Tribe API (External CRM System)](#tribe-api-external-crm-system)
    - [Apsis One Platform (Internal Dependencies)](#apsis-one-platform-internal-dependencies)
  - [11. Procedures & Workflows](#11-procedures-workflows)
    - [Installation Workflow](#installation-workflow)
    - [Form Submission with CRM Sync Workflow](#form-submission-with-crm-sync-workflow)
    - [CRM Data Download Workflow](#crm-data-download-workflow)
    - [Consent Export Workflow](#consent-export-workflow)
    - [Profile Merge Workflow (Within Integration)](#profile-merge-workflow-within-integration)
  - [12. Configuration Reference](#12-configuration-reference)
  - [13. Tech Stack Summary](#13-tech-stack-summary)
  - [14. Open Questions for Follow-up](#14-open-questions-for-follow-up)
  - [15. Change Log / Session Info](#15-change-log-session-info)

---

# Apsis One Integrations — tribe Knowledge Base

## 1. Subdomain Overview

The **tribe** subdomain manages contact and entity record synchronization between Apsis One and the Tribe CRM system. It implements keyspace-based isolation of contact/lead/custom entity records, form submission routing with CRM sync capability, and profile merging workflows. The subdomain handles bidirectional data flow: forms submitted in Apsis route to Tribe with profile identification data, and Tribe entity downloads populate corresponding keyspaces in Apsis. Critical responsibility: prevent data inconsistency by maintaining Tribe as master of contact data and enforcing entity-type-specific keyspace separation to avoid incorrect consent exports.

---

## 2. Architecture Map

```
┌─ Installation Flow ──────────────────────────────────────────────┐
│  Tribe Integration Installed                                     │
│  └─> Installation Manager.main_install()                         │
│      ├─> Create main keyspace (contact entity)                   │
│      ├─> Loop: Create entity-specific keyspaces per Tribe config │
│      │   └─> discriminator: integrations:keyspaces:[hash]:[type] │
│      └─> Store discriminator mappings in DB for lookup           │
│          └─> Register event listeners on form campaigns          │
│              (if 'sync to CRM' enabled)                          │
└──────────────────────────────────────────────────────────────────┘

┌─ Form Submission Flow ───────────────────────────────────────────┐
│  User submits form (sync to CRM enabled)                         │
│  └─> Submit event listener fires                                 │
│      └─> Payload: {profileKey, identifyingData, fields}          │
│          ├─> Email / Phone / CRM ID / Lead ID                    │
│          └─> Send to Tribe API                                   │
│              └─> Tribe responds:                                 │
│                  ├─> entityType (contact|lead|custom)            │
│                  └─> recordId (new or existing)                  │
│                      └─> Lookup keyspace by entity type          │
│                          └─> Store recordId in keyspace          │
│                              └─> IF silhouette/lead:             │
│                                  Merge with contact keyspace     │
└──────────────────────────────────────────────────────────────────┘

┌─ CRM Data Download Flow ────────────────────────────────────────┐
│  Tribe sync scheduled/triggered                                 │
│  └─> Tribe returns entity list (type + id)                      │
│      └─> For each entity:                                       │
│          ├─> Determine entity type (contact|lead|custom)         │
│          ├─> Look up keyspace discriminator in DB                │
│          │   └─> integrations:keyspaces:[hash]:[type]            │
│          ├─> Retrieve keyspace ID from discriminator             │
│          └─> Update profile in entity-specific keyspace          │
└──────────────────────────────────────────────────────────────────┘

┌─ Keyspace Isolation ────────────────────────────────────────────┐
│  Main Keyspace (Contact)                                         │
│  ├─> Entity Keyspace (Lead) – dynamic per Tribe config           │
│  └─> Entity Keyspace (Custom Entity) – if configured             │
│      └─> Separated to prevent:                                   │
│          ├─ Consent exports including wrong entity types         │
│          └─ Data model conflicts                                 │
│          └─ Duplicate profile creation                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Module Reference

### Installation Manager
- **Purpose**: Bootstraps keyspaces and event listeners when Tribe integration is installed on an account section.
- **Key files**: `Installation Manager (main_install function)`
- **Tech stack**: Apsis One Integrations platform
- **Data flow**: 
  - **In**: Installation trigger (integration ID, section ID, CRM logical name)
  - **Out**: Keyspace discriminators stored in DB; event listeners registered
- **Business rules**:
  - Create main keyspace with discriminator format: `integrations:keyspaces:[8-char-hash]:[tribe]`
  - Loop through entity types defined in Tribe configuration (contact, lead, custom entities as per customer selection)
  - For each entity type, generate discriminator and create entity-specific keyspace
  - Store all discriminator→keyspaceId mappings in database for later lookup
  - On reinstallation: reuse existing keyspace (do NOT create new)
- **Configuration**:
  - Entity type mapping: Tribe supports dynamic entity selection (customers choose contact, lead, custom entity via dropdown)
  - Keyspace discriminator format hardcoded; hash derived from section discriminator (first 8 chars)
- **Integration points**: Database (keyspace discriminator storage); Tribe CRM metadata (entity type enumeration)
- **Gotchas**:
  - ⚠️ **Tribe Dynamic Entity Issue**: Tribe's 'lead' entity is not a real entity type in Tribe—it's a virtual representation of the customer's dynamic dropdown selection. Customers can select contact, lead, custom entity ('sailing boats', etc.), but Tribe never responds with the actual selected type. Apsis bootstraps a lead keyspace, but Tribe won't use it.
  - Uninstalling integration does NOT delete keyspaces (intentional; preserves historical data and enables re-engagement)
  - Reinstalling reuses existing keyspace with all historical event data
- **Tribal knowledge**:
  - The choice of 8 characters for hash length is undocumented; used for uniqueness only, not reverse resolution
  - Discriminator reuse on reinstall is deliberate for "win-back" customer re-engagement scenarios

---

### Keyspace System
- **Purpose**: Uniquely identifies and manages contact/lead/custom entity records across a Tribe integration installation, preventing data model conflicts and ensuring correct entity-type routing.
- **Key files**: Managed by Installation Manager; discriminators stored in database
- **Tech stack**: Apsis One Integrations platform (DB layer for discriminator lookup)
- **Data flow**:
  - **In**: Entity type from CRM response or download
  - **Out**: Keyspace ID for use in profile update/query operations
  - Lookup: `entityType + integrationId + sectionId` → discriminator → keyspaceId
- **Business rules**:
  - Discriminator format: `integrations:keyspaces:[8-char-hash]:[tribe]` (+ optional :[entityType] for multi-entity lookups)
  - Keyspace NEVER deleted on uninstall
  - Keyspace reused on reinstall to preserve historical data and event correlation
  - Hash provides uniqueness across multiple Tribe installations on same account without reverse resolution needed
- **Configuration**:
  - Keyspace discriminator mappings table (exact schema not documented; includes discriminator string, keyspaceId, entityType, integrationId, sectionId)
- **Integration points**: Database (discriminator lookup table)
- **Gotchas**:
  - ⚠️ No mechanism to reverse-resolve hash back to section; collision risk across sections is theoretically possible but mitigated by hash approach (collisions unlikely with first 8 chars of UUID-like hash)
  - Discriminators must match exactly; case sensitivity not specified
  - Missing discriminator in DB lookup will fail silently (no error logging documented)
- **Tribal knowledge**:
  - 8-char hash is intentionally non-reversible; design assumes hash collisions are acceptable because section ID is in discriminator anyway
  - Keyspace persistence across uninstall/reinstall is core feature for customer win-back scenarios

---

### Form Sync / Event Listener System
- **Purpose**: Registers event listeners ('start viewed', 'submit') for form campaigns when 'sync to CRM' option is enabled, enabling automatic form submission routing to Tribe.
- **Key files**: Form Campaign Configuration UI; Event Listener Registration (Implementation not specified in transcript)
- **Tech stack**: Apsis One Integrations platform event listener framework
- **Data flow**:
  - **In**: Form published with 'sync to CRM' enabled
  - **Out**: Event listeners registered on Apsis event bus for that form
  - Submit event fires → Listener triggers CRM sync workflow
- **Business rules**:
  - 'sync to CRM' option only visible/enabled if Tribe integration is installed
  - Event listeners removed on integration uninstall (no orphaned listeners)
  - Event listeners only register for campaigns with explicit 'sync to CRM' enablement
  - Under normal circumstances, form submissions never reach uninstalled/inactive integrations
- **Configuration**:
  - 'sync to CRM' toggle per form campaign
  - Event listener events: 'start viewed', 'submit' (at minimum)
- **Integration points**: Form Campaign UI; Event Bus (Apsis internal)
- **Gotchas**:
  - ⚠️ If integration is uninstalled, 'sync to CRM' option disappears from form UI; existing enabled forms silently stop syncing (no error messaging documented)
  - Integration must be installed at time of form publishing for option to appear; install after publishing requires form republish
- **Tribal knowledge**:
  - Safety feature: hiding 'sync to CRM' option when integration not installed prevents orphaned form submissions and confusing behavior

---

### Form Submission Routing
- **Purpose**: Routes form submissions to Tribe API and processes responses, storing returned IDs in correct entity-specific keyspaces; handles multi-entity scenarios (e.g., silhouette → contact merge).
- **Key files**: Form Submission Handler (implementation not specified)
- **Tech stack**: Apsis One Integrations platform; REST client for Tribe API
- **Data flow**:
  - **In**: Form submission event from Apsis form engine; profileKey + identifying data (email, phone, CRM ID, lead ID) + form fields
  - **Out**: HTTP POST to Tribe API endpoint
  - Tribe response: `{entityType, recordId}` (and optional merge metadata)
  - Store recordId in corresponding entity keyspace
- **Business rules**:
  - Form payload structure: `{profileKey, identifyingData: {email, phone, crmId, leadId}, fields: {...}}`
  - profileKey is added by Apsis (identifies profile in Apsis DB)
  - CRM response determines target keyspace via entity type lookup
  - If Tribe returns entity type mismatch (e.g., expected lead, got silhouette), route to correct keyspace (not expected for Tribe, but design handles it)
  - Form submissions only processed if integration is installed and active
- **Configuration**:
  - Identifying data fields (email, phone, etc.) vary by CRM and form configuration
  - fields property contains submitted form field values (structure not fully specified; likely flat object but could be nested)
- **Integration points**: Tribe API (form submission endpoint); Entity Keyspaces (store returned IDs); Profile Merging Engine (if multi-entity response)
- **Gotchas**:
  - ⚠️ **Tribe Dynamic Entity Issue**: Tribe never responds with actual entity type from dynamic dropdown. Form submission might be expected as 'lead' but Tribe responds with whatever customer selected (contact, custom entity, etc.). Routing logic must handle whatever Tribe returns, not what form configuration expects.
  - If integration uninstalled after form publishes, 'sync to CRM' listeners removed and form submissions continue without CRM sync (no error raised to user)
  - Entity type mismatch between configuration and Tribe response could cause ID storage in wrong keyspace if routing logic doesn't validate
- **Tribal knowledge**:
  - Tribe's dynamic dropdown makes form submission routing inherently unpredictable—form might be configured as lead but route to contact keyspace depending on customer's Tribe UI selection

---

### Entity-Specific Keyspaces
- **Purpose**: Separate storage containers for different entity types (contact, lead, custom entities) to prevent data model conflicts and incorrect consent exports.
- **Key files**: Installation Manager (entity for loop); Form Submission Handler (entity type lookup)
- **Tech stack**: Apsis One Integrations platform (keyspace storage layer)
- **Data flow**:
  - **In**: Entity type identifier (contact|lead|customEntity) from CRM response or download
  - **Out**: Keyspace ID for profile update/query
  - Lookup chain: entityType → discriminator pattern → DB lookup → keyspaceId
- **Business rules**:
  - Main contact keyspace created for all integrations
  - Additional entity keyspaces created per Tribe configuration during installation (e.g., lead, custom entity)
  - Silhouettes/leads never stored in contact keyspace (prevents incorrect consent exports)
  - Contact records never merged with external keyspace profiles (e.g., email keyspace) within Apsis; CRM is master of merge decisions
  - Entity type from CRM response determines keyspace routing; no aliasing or fallback
- **Configuration**:
  - Tribe supports dynamic entity selection: customers choose primary and additional entities during setup
  - Entity keyspace naming convention not specified (likely discriminator-based)
- **Integration points**: Keyspace System (discriminator lookup); Form Submission Handler (entity type routing); CRM Data Download (entity type from Tribe response)
- **Gotchas**:
  - ⚠️ If silhouette/lead ID incorrectly stored in contact keyspace, consent exports will include that ID in payloads back to Tribe, causing unexpected data sync issues
  - ⚠️ Tribe's dynamic entity dropdown means entity types are customer-configurable and not guaranteed at runtime
  - If keyspace discriminator lookup fails, no error handling documented; operation may fail silently
- **Tribal knowledge**:
  - Entity keyspace separation is critical for preventing data corruption in consent exports
  - Silhouette/lead keyspaces act as holding areas for temporary records; merge into contact only when CRM signals conversion

---

### Profile Merging Engine
- **Purpose**: Merges lead/custom entity records with contact records when Tribe indicates conversion or form submission creates multiple entity types.
- **Key files**: Profile Merging Logic (implementation not specified; likely in Form Submission Handler or separate merge module)
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Merge request signal (from Tribe response or form submission with multi-entity payload); source entity ID (lead/silhouette) + destination profile key (contact)
  - **Out**: Merged profile record linking silhouette ID with contact profile key
- **Business rules**:
  - Merge only occurs within integration entity keyspaces (e.g., lead ↔ contact keyspaces)
  - NEVER merge CRM-originated contacts with external keyspace profiles (e.g., email keyspace); CRM must be master
  - Merge prevents duplicate profile creation if same person resubmits form with same email
  - Merges initiated outside integration (e.g., in Apsis UI) will NOT propagate back to Tribe and will cause data inconsistency
  - Tribe should be master of merge decisions; Apsis executes merge instructions, not initiates them
- **Configuration**: Merge rules per entity type pair (e.g., lead → contact, custom entity → contact)
- **Integration points**: Entity Keyspaces (source and destination); Form Submission Routing (merge trigger)
- **Gotchas**:
  - ⚠️ Merging CRM contacts with other keyspaces in Apsis UI is possible but prohibited; changes won't sync back to Tribe
  - ⚠️ If merge request from Tribe never arrives (e.g., due to API timeout), silhouette/lead remains orphaned in separate keyspace; no automatic cleanup documented
  - ⚠️ Historical event data association after merge not specified; querying merged profiles might not surface all events
- **Tribal knowledge**:
  - Merge is designed as holding pattern: silhouette/lead keyspace temporarily stores record until Tribe confirms permanent entity (contact), then merge consolidates data
  - CRM must be master of merge decisions; Apsis design prevents user-initiated merges from breaking Tribe sync

---

### CRM ID Management
- **Purpose**: Track and store Tribe record IDs (from form submissions and downloads) in appropriate entity keyspaces; enable future profile lookups and updates via Tribe ID.
- **Key files**: Entity Keyspaces; Form Submission Routing; CRM Data Download
- **Tech stack**: Apsis One Integrations platform (keyspace storage)
- **Data flow**:
  - **In**: Record ID from Tribe API response (form submission or entity download)
  - **Out**: Stored in entity-specific keyspace; available for future lookups
  - Lookup: Apsis profile key → Tribe ID lookup → retrieve record from Tribe
- **Business rules**:
  - Tribe record IDs stored in entity-specific keyspace matching entity type in Tribe response
  - Record ID unique per entity type per Tribe installation
  - Profile must have either Tribe ID (if CRM-originated) or profile key (if Apsis-native); mapping enables bidirectional sync
  - Tribe ID never updated to point to different Tribe record (1:1 mapping maintained)
- **Configuration**: Record ID storage location determined by entity type discriminator lookup
- **Integration points**: Entity Keyspaces; Form Submission Routing; CRM Data Download
- **Gotchas**:
  - ⚠️ If Tribe returns entity type mismatch, ID stored in wrong keyspace, breaking future lookups
  - ⚠️ No deduplication if form submitted twice with same email; depends on Tribe to handle duplicate detection and return existing ID
  - ⚠️ **Tribe Dynamic Entity**: Tribe may not return actual lead ID if customer selected different entity type in dropdown
- **Tribal knowledge**:
  - ID storage is asymmetrical: Apsis stores Tribe IDs, but Tribe stores Apsis profile keys; merge happens via profile key, not Tribe ID

---

## 4. Cross-Cutting Concerns

### Authentication
- ⚠️ **Not documented in transcript**: Tribe API authentication method not specified (likely OAuth, API key, or custom token; requires verification in Tribe connector code)
- Integration credentials stored securely during installation (location/mechanism not specified)

### Error Handling
- **Form submission failures**: If Tribe API returns error, handling mechanism not documented
  - Likely behavior: form submission fails silently or returns generic error to user
  - No retry logic documented
  - No fallback to alternative entity type
- **Keyspace discriminator lookup failures**: No error handling documented; operation may fail silently
- **Missing entity type**: If Tribe response entityType not recognized, routing may fail or default to contact keyspace (behavior not specified)
- **Network timeouts**: Merge request from Tribe may not arrive if connection dropped; orphaned records possible

### Logging
- ⚠️ **Not documented**: Log levels, trace IDs, debugging info for form submissions and keyspace lookups not specified
- Likely areas: form submission events, keyspace lookups, merge operations, consent exports

### Deployment
- Form sync feature availability tied to integration installation status
- Event listener registration/removal tied to integration lifecycle (install/uninstall)
- Keyspace reuse on reinstall requires database consistency (discriminator mappings must survive uninstall)

### Consent Exports
- **CRITICAL**: Silhouette/lead IDs must NEVER be included in consent exports back to Tribe
- If silhouette ID incorrectly stored in contact keyspace, consent export payload will include it, causing data corruption in Tribe
- Consent export filtering logic must validate entity type before inclusion

---

## 5. Business Rules Reference

### Entity Separation & Data Isolation
1. **Contact/Lead/Custom Entity Keyspace Separation**
   - Contacts, leads, and custom entities stored in separate keyspaces
   - Rationale: Prevent data model conflicts; consent exports must exclude non-contact entities
   - Enforcement: Entity type lookup determines keyspace routing

2. **Silhouette/Lead Holding Pattern**
   - Silhouettes/leads remain separate from contact keyspace until CRM signals merge (conversion)
   - Rationale: Prevent duplicate profile creation if same person resubmits form with same email
   - Enforcement: Merge only occurs on Tribe signal, not Apsis-initiated

3. **No Cross-Keyspace Merges of CRM-Originated Contacts**
   - CRM-originated contacts must NOT be merged with external keyspace profiles (e.g., email keyspace) within Apsis
   - Rationale: CRM is master of contact data; merges outside Apsis won't propagate back to Tribe
   - Enforcement: Form submission merges only within integration entity keyspaces; UI merges prohibited

### CRM as Master
4. **CRM Master of Contact Data**
   - Tribe system is authoritative source for contact/lead/entity data
   - Apsis changes to CRM-originated profiles may not sync back to Tribe
   - Rationale: Bidirectional sync complexity; CRM remains source of truth

5. **CRM Master of Merge Decisions**
   - Silhouette→Contact conversions, Lead→Contact promotions initiated by Tribe, not Apsis
   - Tribe response triggers merge; Apsis executes, does not initiate
   - Rationale: Prevent data inconsistency; CRM logic determines when conversion occurs

### Data Persistence
6. **Keyspace Preservation on Uninstall**
   - Keyspaces NEVER deleted when integration uninstalled
   - Historical event data and contact profiles remain in Apsis
   - Rationale: Enable customer re-engagement campaigns; GDPR cleanup not tied to uninstall (separate flow)

7. **Keyspace Reuse on Reinstall**
   - Reinstalling Tribe integration reuses existing keyspace with all historical data
   - Rationale: Preserve event correlation; enable win-back scenarios

### Form Sync Features
8. **'sync to CRM' Option Visibility**
   - Form 'sync to CRM' option only visible if Tribe integration installed
   - Option must be explicitly enabled per form
   - Rationale: Prevent accidental routing to inactive integrations; under normal circumstances, form submissions never reach uninstalled integrations

9. **Form Submission Routing Constraints**
   - Form submissions only sent to Tribe if 'sync to CRM' enabled AND integration installed
   - Form data payload includes profileKey, identifying data (email, phone, CRM ID), and form fields
   - Tribe response determines target entity keyspace via entity type
   - Rationale: Safety mechanism; ensures form submissions don't orphan in uninstalledintegrations

### Consent & Export
10. **Consent Export Entity Filtering**
    - Consent exports back to Tribe must include ONLY contact records
    - Silhouette, lead, custom entity IDs must be excluded
    - Rationale: Tribe expects contact consent; including other entity types causes data corruption

---

## 6. Known Issues & Workarounds

### Issue 1: Tribe Dynamic Entity Dropdown Behavior
- **Description**: Tribe's 'lead' entity is virtual. Customers can select contact, lead, or custom entities via dynamic dropdown. Apsis bootstraps lead keyspace during installation, but Tribe never responds with actual 'lead' type—response reflects customer's dropdown selection. Form submission routing expects one entity type but receives another.
- **Severity**: Medium
- **Affected Components**: Installation Manager, Form Submission Routing, Entity Keyspaces, Tribe Integration
- **Workaround**: Routing logic must accept whatever entity type Tribe returns and route accordingly. No validation of expected vs. actual type.
- **Long-term**: Erik Johansson attempting to simplify behavior, but no timeline committed. May require Tribe to expose entity type metadata during installation.
- **Impact on tribe**: Form submissions may route to unexpected keyspaces if customer changes Tribe dropdown selection after installation.

### Issue 2: Custom Keyspace Configuration (JoinCX precedent, applies to Tribe)
- **Description**: Customer requested per-installation custom keyspace configuration (e.g., use email keyspace instead of CRM keyspace). Code-based solution would apply to all customers of that CRM connector, not just requesting customer. System not designed for per-customer keyspace customization.
- **Severity**: Low
- **Affected Components**: Installation Manager, Keyspace System
- **Workaround**: None documented. Wednesday follow-up scheduled between Erik and Lukasz to explore non-code solutions (e.g., database-driven configuration, account-level settings).
- **Impact on tribe**: If Tribe customer requests custom keyspace behavior, no currently-available solution; code change would affect all Tribe customers.

### Issue 3: No Reverse Resolution of Keyspace Discriminator Hash
- **Description**: 8-character hash in discriminator is non-reversible. If hash collision occurs across different account sections, no mechanism to identify source section.
- **Severity**: Low
- **Affected Components**: Keyspace Discriminator Generation, Keyspace System
- **Workaround**: Collision risk mitigated by design (hash-based uniqueness deemed acceptable; collisions unlikely). No need for reverse resolution.
- **Impact on tribe**: Minimal risk; if collision occurs, discriminator lookup would return wrong keyspace, causing data in wrong keyspace (silent failure).

### Issue 4: Orphaned Silhouette/Lead Records
- **Description**: If merge request from Tribe never arrives (API timeout, crash), silhouette/lead record remains orphaned in separate keyspace indefinitely. No automatic cleanup or archiving.
- **Severity**: Low
- **Affected Components**: Profile Merging Engine, Entity Keyspaces
- **Workaround**: Manual intervention (not documented). Tribe should retry merge request on resync.
- **Impact on tribe**: Orphaned records accumulate over time; unclear if they affect profile deduplication or event querying.

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Keyspace** | Unique identifier and storage container for a specific entity type (contact, lead, custom entity) within a Tribe integration on a specific account section. Identified by discriminator: `integrations:keyspaces:[8-char-hash]:[tribe]` |
| **Discriminator** | Unique string identifier for a keyspace: `integrations:keyspaces:[8-char-hash]:[tribe]` (+ optional :[entityType]). Used for database lookup to retrieve keyspace ID. |
| **Entity Type** | Category of record in Tribe (contact, lead, custom entity). Determines which keyspace stores the record. Dynamic per customer in Tribe. |
| **Silhouette** | Temporary/incomplete contact record created from form submission before becoming full contact. Tribe-specific terminology (replaces "lead" in some contexts). Stored in separate keyspace. |
| **Lead** | Potential customer record in Tribe; may be converted to contact. Virtual in Tribe's case (represents dropdown selection, not actual entity). Stored in separate keyspace from contacts. |
| **Custom Entity** | Customer-defined entity type in Tribe (e.g., 'sailing boats'). Tribe's dynamic dropdown feature. Stored in separate keyspace. |
| **Main Keyspace** | Primary keyspace created for Tribe integration, typically for contact entity type. Created first during installation. |
| **Entity Keyspace** | Keyspace dedicated to specific entity type (lead, silhouette, custom entity) beyond main contact keyspace. Created during installation per Tribe configuration. |
| **Form Sync** | Feature enabling form submissions to route to Tribe. Must be explicitly enabled per form; only visible if integration installed. Also called "sync to CRM" option. |
| **Event Listener** | Mechanism registering for form events ('start viewed', 'submit') when sync to CRM enabled. Triggers Tribe sync. Removed on integration uninstall. |
| **Profile Key** | Unique Apsis identifier for a profile/contact. Added to form submission payloads. Used for profile lookup in Apsis DB. |
| **CRM ID / Record ID** | Unique identifier for record in Tribe. Returned in form submission response or entity download. Stored in entity-specific keyspace. |
| **Merge Request** | Signal from Tribe to merge two related entities (e.g., silhouette → contact conversion). Triggers profile merge within integration keyspaces. |
| **Export Keyspace** | Keyspace containing profile key; used in merges to tie silhouette ID with contact profile key information. |
| **Master Data / CRM is Master** | Design principle: Tribe is authoritative source of contact data. Apsis should not override Tribe data. Merges initiated by Tribe, not Apsis. |
| **Consent Export** | Process sending contact consent changes back to Tribe. Critical sensitivity: silhouette/lead IDs must NOT be included (prevents data corruption in Tribe). |
| **Win-Back Scenario** | Re-engagement campaign targeting customers from previous campaigns. Enabled by keyspace preservation on uninstall/reinstall. |
| **GDPR Cleanup** | Explicit deletion of contact records upon data subject request. Separate from integration uninstallation; does delete keyspace data. |
| **Discrimination / Discriminator** | Method of uniquely identifying keyspaces via formatted string. Discriminator = integrations:keyspaces:[hash]:[crm-name]. |
| **Hash (8-char)** | First 8 characters of section discriminator used in keyspace discriminator for uniqueness. Non-reversible; collisions unlikely. |

---

## 8. Confidence Notes

### High Confidence ✓
- Keyspace discriminator format and rationale
- Entity keyspace separation requirement and rationale
- CRM-as-master principle and enforcement
- Form sync feature visibility and triggers
- Keyspace preservation on uninstall and reuse on reinstall
- Business rules around consent exports
- Profile merging rules (within integration only)

### Medium Confidence ⚠️
- Exact database schema for keyspace discriminator mapping (structure inferred, not documented)
- Tribe entity type enumeration and how it's determined during installation (likely hardcoded or queried, not specified)
- Form submission payload structure (assumes profileKey + identifyingData + fields; nested field structure unclear)
- Event listener implementation details (event names 'start viewed', 'submit' mentioned but other events may exist)
- Error handling for form submissions (behavior on Tribe API failure not documented)
- Logging and debug information (locations and levels not specified)
- Tribe API authentication method (mechanism not specified in transcript)

### Low Confidence / Unresolved ❌
- Why specifically 8 characters for hash? (arbitrary or performance-based unknown)
- How entity types determined for Tribe configuration during installation? (hardcoded, queried, or user-selected unknown)
- Per-customer keyspace customization solution for JoinCX precedent (non-code solution investigation ongoing)
- Exact format of form submission payload sent to Tribe API (fields structure not fully specified)
- Handling of Tribe response timeouts and merge request failures (retry logic, cleanup not documented)
- Automatic cleanup/archiving of orphaned silhouette/lead records (no mechanism documented)
- Historical event data querying and consent tracking after profile merge (side effects unknown)
- Full schema of keyspace discriminator mapping table (inferred: discriminator, keyspaceId, entityType, integrationId, sectionId)

### Conflicts/Disagreements
- **Tribe 'lead' entity reality**: Transcript indicates 'lead' is virtual (represents dropdown), not actual entity type. Apsis bootstraps lead keyspace but Tribe may never use it. This is foundational to understanding Tribe complexity but could benefit from clarification from Tribe team directly.

### Gaps
- No documented integration between Tribe metadata API and Installation Manager (how does Apsis learn what entity types Tribe supports?)
- No documented handling of Tribe API rate limits or throttling
- No documented behavior if form submitted while Tribe integration is in process of being uninstalled
- No documented behavior for concurrent form submissions with same email during silhouette→contact conversion window
- No documented GDPR/data deletion flow integration with Tribe (separate from uninstall)

---

## 9. Tribal Knowledge (Non-Obvious Domain Facts)

1. **Keyspace Hash is Intentionally Non-Reversible**
   - The 8-character hash in discriminator is NOT meant to be reverse-resolved. You don't need to (and shouldn't) figure out which section a hash came from. Design assumes no reverse resolution needed because section ID is already in discriminator context.
   - *Why matters*: Don't waste time trying to reverse-engineer hash collisions; design accepts them as acceptable risk.

2. **Uninstall ≠ Delete**
   - Uninstalling Tribe integration DOES NOT delete keyspace. Counterintuitive. Historical event data remains in Apsis. If customer reinstalls Tribe, old keyspace is reused with all its data.
   - *Why matters*: Enables "win-back" re-engagement campaigns. If customer pauses Tribe integration for 6 months, then reinstalls, all their profile history is still there. This is intentional and critical for customer retention campaigns.

3. **Tribe's Lead is a Virtual Ghost**
   - Tribe doesn't actually have a "lead" entity type in the way Microsoft Dynamics does. In Tribe's UI, customers can select what entity type to use (contact, lead, custom entity like 'sailing boats'). This is a dynamic dropdown. Apsis bootstraps a lead keyspace during installation, but Tribe may never respond with type='lead' if customer selected something else.
   - *Why matters*: Form submission routing is unpredictable. You might expect lead keyspace to be populated, but it sits empty because customer's Tribe UI is configured differently. This is the #1 source of confusion in Tribe integration.

4. **Silhouette Keyspace is a Holding Area**
   - In E-deal context (mentioned in discussion), silhouettes are temporary records waiting for Tribe confirmation. They live in a separate keyspace until Tribe confirms conversion to contact (merge request). This prevents duplicates if same person submits form twice with same email.
   - *Why matters*: Don't try to merge silhouettes with contacts in Apsis UI. Tribe owns the merge decision. If you merge in Apsis, Tribe won't know about it and will create duplicates on next sync.

5. **Consent Exports are Silhouette Mines**
   - If you accidentally store a silhouette/lead ID in the contact keyspace, consent export will include it in payloads back to Tribe. Tribe will then export that non-contact record as if it were a contact, corrupting Tribe's database. CRITICAL mistake.
   - *Why matters*: Double-check entity type before storing in contact keyspace. Entity-specific keyspace separation is not just cleanliness; it's data integrity protection.

6. **Form Sync Invisibility is a Safety Feature**
   - If Tribe integration is not installed, the 'sync to CRM' option literally doesn't appear in form configuration UI. Under normal circumstances, form submissions never reach uninstalled integrations.
   - *Why matters*: Customers can't accidentally enable form routing to inactive integrations. Form data won't orphan or bounce.

7. **CRM is Master, Apsis is Servant**
   - Tribe is source of truth for contact data. Changes you make in Apsis to CRM-originated profiles may not sync back. Merges should happen in Tribe first, then sync to Apsis.
   - *Why matters*: Never merge CRM contacts with Apsis-native profiles in Apsis. The merge won't propagate back and Tribe will become out-of-sync, creating duplicates.

8. **Keyspace Lookup is Bidirectional**
   - Form submission includes profileKey (Apsis side) and identifying data (email, phone, Tribe ID). Tribe responds with recordId. Apsis stores Tribe ID in keyspace. Future downloads use Tribe ID to find records. Merge uses profileKey to tie them together.
   - *Why matters*: Understanding the asymmetry (Apsis stores Tribe IDs, Tribe stores Apsis profile keys) explains how merges work and why entity type matters.

9. **8 Characters is Arbitrary (Probably)**
   - Why 8 chars for hash? No documentation. Likely arbitrary choice or performance-based. But it's the rule now and changing it would require migration.
   - *Why matters*: Don't second-guess it. Don't try to optimize it. It works. Collisions unlikely enough for practical purposes.

10. **Per-Customer Customization is Governance Nightmare**
    - Customer requested custom keyspace behavior for JoinCX. Code-based solution would apply to ALL JoinCX customers, not just requester. System isn't built for per-customer feature toggles at this level.
    - *Why matters*: Understand the organizational pain point. Feature requests that favor one customer at expense of others are hard to justify. This is why it's still unresolved.

---

## 10. Integration Points Detail

### Tribe API (External CRM System)

**Form Submission Endpoint**
- **Direction**: Apsis → Tribe
- **Protocol**: REST/HTTP (method and exact URL not specified)
- **Payload**: 
  ```json
  {
    "profileKey": "uuid",
    "identifyingData": {
      "email": "user@example.com",
      "phone": "+1234567890",
      "crmId": "tribe-contact-id",
      "leadId": "tribe-lead-id"
    },
    "fields": {
      "field1": "value1",
      "field2": "value2"
    }
  }
  ```
- **Response**: 
  ```json
  {
    "entityType": "contact|lead|custom",
    "recordId": "tribe-record-id"
  }
  ```
- **Authentication**: Not documented (likely OAuth, API key, or custom token)
- **Frequency**: Per form submission with sync enabled
- **Error Handling**: Not documented; form submission may fail silently

**Entity Download Endpoint**
- **Direction**: Tribe → Apsis
- **Protocol**: REST/HTTP (method and exact URL not specified) or Webhook
- **Payload**: Entity list with type and ID for each record
- **Frequency**: Scheduled or event-triggered (frequency not specified)
- **Handling**: For each entity, lookup keyspace by type, update profile in correct keyspace

**Consent Export Endpoint**
- **Direction**: Apsis → Tribe
- **Protocol**: REST/HTTP (method and exact URL not specified)
- **Payload**: Contact consent changes (must exclude silhouette/lead IDs)
- **Frequency**: Per consent change (varies by Apsis configuration)
- **Critical**: Entity type validation required before inclusion

### Apsis One Platform (Internal Dependencies)

**Database Layer** (Keyspace Discriminator Storage)
- Store/retrieve discriminator → keyspaceId mappings
- Schema not fully documented; inferred fields: discriminator, keyspaceId, entityType, integrationId, sectionId
- Persistence required across uninstall/reinstall

**Event Bus** (Form Event Listeners)
- Register/deregister listeners for form submission events
- Event types: 'start viewed', 'submit' (at minimum)
- Listeners tied to integration lifecycle

**Profile Management** (Keyspace Storage)
- Update/query profiles by keyspace ID
- Entity-type-specific validation (don't store silhouette ID in contact keyspace)
- Merge operations between keyspaces

**Form Configuration UI**
- Conditional visibility of 'sync to CRM' option based on integration install status
- Form field mapping to Tribe entity fields

**Consent Export System**
- Query contact records from keyspace
- Filter by entity type (contacts only for Tribe)
- Send to Tribe API

---

## 11. Procedures & Workflows

### Installation Workflow
```
Tribe Integration Installation
├─ Installation triggered: (integrationId, sectionId, crmLogicalName='tribe')
├─ Installation Manager.main_install() called
├─ Main keyspace created
│  ├─ Discriminator: integrations:keyspaces:[8-char-hash]:[tribe]
│  └─ Stored in DB
├─ Loop through entity types (contact, lead, custom entities per Tribe config)
│  └─ For each entity type:
│     ├─ Generate entity-specific discriminator: .../[tribe]:[entityType]
│     ├─ Create keyspace
│     └─ Store mapping in DB
├─ Event listener registration
│  └─ For each form with 'sync to CRM' enabled:
│     ├─ Register 'start viewed' listener
│     └─ Register 'submit' listener
└─ Installation complete
   
Reinstallation After Uninstall
├─ Uninstall does NOT delete keyspace
├─ Reinstall reuses existing keyspace discriminators from DB
├─ No data loss; historical events preserved
└─ Reinstallation complete
```

### Form Submission with CRM Sync Workflow
```
Form Submission → CRM Sync
├─ User fills form, clicks submit
├─ Form submit event triggered in Apsis
├─ Event listener fires (only if 'sync to CRM' enabled AND integration installed)
├─ Collect form submission data:
│  ├─ profileKey (Apsis profile ID)
│  ├─ identifyingData: {email, phone, crmId, leadId}
│  └─ fields: {form field values}
├─ POST to Tribe API form submission endpoint
│  └─ Wait for response: {entityType, recordId}
├─ Look up correct keyspace by entityType
│  ├─ Query DB: WHERE discriminator = integrations:keyspaces:...:tribe:[entityType]
│  └─ Retrieve keyspaceId
├─ Store recordId in entity-specific keyspace
│  └─ Update profile with Tribe ID
├─ IF entityType is lead/silhouette:
│  ├─ Trigger merge workflow
│  └─ Merge silhouette ID + profileKey in contact keyspace
└─ Form submission complete; record now synced to Tribe

Error Cases (Not Fully Documented)
├─ Integration uninstalled → 'sync to CRM' option hidden before form submission
├─ Tribe API unreachable → Form submission fails (silently?)
├─ entityType not recognized → Keyspace lookup fails (silent failure?)
└─ Merge request never arrives → Silhouette remains orphaned indefinitely
```

### CRM Data Download Workflow
```
Tribe → Apsis Data Sync
├─ Scheduled or event-triggered sync initiated
├─ Tribe API returns entity list: [{type: contact|lead|custom, id: record-id}, ...]
├─ For each entity in list:
│  ├─ Determine entityType (contact|lead|custom)
│  ├─ Look up keyspace discriminator in DB:
│  │  ├─ Query: WHERE integration='tribe' AND section=sectionId AND entityType=[type]
│  │  └─ Retrieve discriminator
│  ├─ Retrieve keyspaceId from discriminator
│  └─ Update/create profile in keyspace with Tribe data
└─ Sync complete

Error Cases
├─ Entity type not in DB → Keyspace lookup fails (data not synced)
├─ Keyspace deleted (shouldn't happen) → No destination for profile
└─ Network timeout → Retry mechanism (not documented)
```

### Consent Export Workflow
```
Consent Change → Tribe Export
├─ Contact consent changed in Apsis
├─ Identify affected profiles from contact keyspace
├─ Query: SELECT profiles FROM keyspace WHERE keyspaceId=[contact-keyspace-id]
├─ For each profile:
│  ├─ Validate: entity type = contact (NOT lead, silhouette, custom)
│  │  └─ ⚠️ CRITICAL: Silhouette ID in contact keyspace = data corruption
│  ├─ Retrieve Tribe ID from profile
│  └─ Include in consent export payload
├─ POST to Tribe consent export endpoint
│  └─ Tribe updates contact records with new consent values
└─ Export complete

Error Cases
├─ Silhouette ID incorrectly in contact keyspace → Silhouette included in export → Tribe data corrupted
└─ Tribe API unreachable → Consent export fails (retry mechanism not documented)
```

### Profile Merge Workflow (Within Integration)
```
Silhouette → Contact Merge (E-deal/Tribe Scenario)
├─ Form submission received with silhouette data
├─ Tribe responds: {entityType: 'silhouette', recordId: 'sil-123'}
├─ Store silhouette ID in silhouette keyspace
├─ Trigger merge: silhouette ID + profileKey (contact keyspace)
│  ├─ Look up silhouette keyspace: integrations:keyspaces:...:tribe:silhouette
│  ├─ Look up contact keyspace: integrations:keyspaces:...:tribe:contact
│  ├─ Merge silhouette ID into contact profile record
│  └─ Result: contact profile now linked to both silhouette and contact IDs
├─ If same person resubmits form:
│  ├─ Tribe returns silhouette ID again
│  └─ Apsis finds existing merge, updates silhouette record instead of creating duplicate
└─ When Tribe signals conversion:
   ├─ Merge request arrives: silhouette-id → contact-id
   └─ Contact record updated with permanent contact ID

⚠️ Prohibited Merge (Apsis UI)
├─ User manually merges CRM contact with Apsis-native email profile
├─ Merge occurs in Apsis DB
├─ Tribe never learns of merge
└─ Result: Data inconsistency, duplicates on next sync, loss of trust
```

---

## 12. Configuration Reference

| Setting | Value/Pattern | Location | Notes |
|---------|---------------|----------|-------|
| **Keyspace Discriminator Format** | `integrations:keyspaces:[8-char-hash]:[tribe]` | DB storage | Hash = first 8 chars of section discriminator |
| **Entity Type Keyspace Discriminator** | `integrations:keyspaces:[8-char-hash]:[tribe]:[entityType]` | DB storage | Examples: `:tribe:contact`, `:tribe:lead`, `:tribe:custom` |
| **Form Sync Visibility** | Only if integration installed | Form Config UI | Prevents routing to inactive integrations |
| **Event Listener Events** | 'start viewed', 'submit' | Form Campaign Config | Registers on form publish with sync enabled |
| **Keyspace Reuse on Reinstall** | Existing keyspace reused (not recreated) | Installation Manager | Preserves historical data |
| **Keyspace Deletion on Uninstall** | Never deleted | Installation Manager | Historical data preserved intentionally |
| **Form Submission Payload** | `{profileKey, identifyingData, fields}` | Form Submission Handler | identifyingData = {email, phone, crmId, leadId} |
| **Entity Types Supported by Tribe** | contact, lead, custom (dynamic) | Installation Manager | Dynamic per customer; 'lead' is virtual |
| **Merge Scope** | Within integration entity keyspaces only | Profile Merging Engine | NO merges with external keyspaces (email, etc.) |
| **Consent Export Filtering** | Include contacts ONLY | Consent Export System | Exclude silhouettes, leads, custom entities |
| **Hash Length** | 8 characters | Discriminator Generation | Arbitrary choice (probably); undocumented rationale |

---

## 13. Tech Stack Summary

| Component | Technology | Notes |
|-----------|-----------|-------|
| **Integration Platform** | Apsis One Integrations framework | Core runtime environment |
| **API Communication** | REST/HTTP | Direction: Apsis ↔ Tribe (specific methods not documented) |
| **Database** | Apsis One internal DB | Stores keyspace discriminator mappings; schema not fully documented |
| **Event System** | Apsis event listener framework | Form submission events, integration lifecycle events |
| **Authentication** | ⚠️ Not documented | Likely OAuth, API key, or custom token; requires verification |
| **Data Storage** | Keyspace storage layer | Entity-type-specific isolation |
| **Form Engine** | Apsis form builder/runtime | Conditional UI, event triggering |

---

## 14. Open Questions for Follow-up

1. **Installation Entity Type Enumeration**: How does Apsis determine which entity types Tribe supports during installation? Hardcoded, queried from Tribe API, or user-selected during setup?

2. **Tribe Entity Type Resolution**: How does Apsis handle Tribe's dynamic dropdown entity selection? Is there a Tribe metadata API that reveals customer's selection, or does Apsis learn from form submission responses?

3. **Hash Collision Risk**: Has collision analysis been performed on 8-character hash across live account sections? What is acceptable collision probability?

4. **Error Handling for Form Submissions**: What happens when Tribe API returns 500/timeout? Does form fail to user? Retry queue? Silent failure? Log entry?

5. **Merge Request Retry Logic**: If Tribe sends merge request but Apsis connection drops, does Tribe retry? Is there a reconciliation mechanism?

6. **Orphaned Record Cleanup**: Is there a maintenance job to detect and handle silhouettes that never receive merge confirmation? What is acceptable orphan age before action?

7. **Per-Customer Keyspace Config**: What is the non-code solution being explored for custom keyspace requirements? Timeline for JoinCX customer?

8. **Form Field Structure**: Can form fields be nested JSON, or must they be flat? Are there size limits on field payloads?

9. **Tribe Authentication**: What is the exact authentication method (OAuth, API key, custom)? Is it account-level or section-level?

10. **GDPR Deletion Integration**: How does explicit GDPR data deletion interact with keyspace preservation on uninstall? Does GDPR deletion remove keyspace data?

---

## 15. Change Log / Session Info

- **Knowledge Base Created**: From 1 KT session transcript "Keyspaces in Integrations part 2.md"
- **Session Date**: Inferred as 2026-03-25 (extraction timestamp)
- **Covered Topics**: Keyspace System, Installation Manager, Entity-Specific Keyspaces, Form Sync, Profile Merging, CRM ID Management, Tribe-specific behavior
- **Speakers**: Likely product/engineering domain experts (names not extracted from transcript)
- **Next Steps**: Identified unresolved questions requiring follow-up with Tribe team and engineering leadership

---

**This knowledge base is current as of the extraction date and represents high-confidence consensus on tribe subdomain architecture. Flag any discrepancies or outdated information for follow-up sessions.**