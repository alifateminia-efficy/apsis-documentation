---
title: Apsis One Integrations — e-deal (efficy corporate) Knowledge Base
subdomain: e-deal (efficy corporate)
generated: 2026-03-25T12:40:46.851Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (2 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — e-deal (efficy corporate) Knowledge Base](#apsis-one-integrations-e-deal-efficy-corporate-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Installation Manager](#installation-manager)
    - [Keyspace System](#keyspace-system)
    - [Discriminator Generation Function](#discriminator-generation-function)
    - [Generic Connector (E-deal Implementation)](#generic-connector-e-deal-implementation)
    - [Base Installer](#base-installer)
    - [E-deal/FCC Corporate Connector](#e-dealfcc-corporate-connector)
    - [Form Submission Routing Handler](#form-submission-routing-handler)
    - [Form Sync Event Listener System](#form-sync-event-listener-system)
    - [Entity-Specific Keyspaces](#entity-specific-keyspaces)
    - [Profile Merging Engine](#profile-merging-engine)
    - [CRM Data Download Handler (Person Entities)](#crm-data-download-handler-person-entities)
    - [Consent Export Handler](#consent-export-handler)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling & Logging](#error-handling-logging)
    - [Deployment & Lifecycle](#deployment-lifecycle)
    - [Data Consistency & Master Data](#data-consistency-master-data)
    - [Form Submission Safety](#form-submission-safety)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Entity Type Management](#entity-type-management)
    - [Profile Updates & Synchronization](#profile-updates-synchronization)
    - [Keyspace & Multi-Tenancy](#keyspace-multi-tenancy)
    - [Form Submission & Lead Creation](#form-submission-lead-creation)
    - [Merging & Data Integrity](#merging-data-integrity)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [⚠️ Hash Algorithm Uncertainty](#-hash-algorithm-uncertainty)
    - [Silhouette Entity Type Handling (E-deal Specific)](#silhouette-entity-type-handling-e-deal-specific)
    - [Profile Update Flow from E-deal to Apsis](#profile-update-flow-from-e-deal-to-apsis)
    - [Account Section Configuration Structure](#account-section-configuration-structure)
    - [Keyspace Discriminator Database Lookup Failure](#keyspace-discriminator-database-lookup-failure)
    - [Lead-to-Person Conversion Workflow](#lead-to-person-conversion-workflow)
    - [Silhouette/Lead ID Deletion Handling](#silhouettelead-id-deletion-handling)
    - [Section Discriminator Stability](#section-discriminator-stability)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence](#high-confidence)
    - [Medium Confidence](#medium-confidence)
    - [Low Confidence / Unconfirmed](#low-confidence-unconfirmed)
    - [Discrepancies & Conflicts](#discrepancies-conflicts)
    - [Outdated / Superseded Information](#outdated-superseded-information)
  - [9. Tribal Knowledge (Non-Obvious Facts)](#9-tribal-knowledge-non-obvious-facts)
    - [Architectural Decisions](#architectural-decisions)
    - [Business Process Insights](#business-process-insights)
    - [Implementation Gotchas](#implementation-gotchas)
    - [Design Principles](#design-principles)
  - [10. Contact & Authority References](#10-contact-authority-references)
  - [11. Quick Reference: E-deal Integration Flow Diagram](#11-quick-reference-e-deal-integration-flow-diagram)
  - [12. Key Takeaways for AI Coding Agent](#12-key-takeaways-for-ai-coding-agent)

---

# Apsis One Integrations — e-deal (efficy corporate) Knowledge Base

## 1. Subdomain Overview

The **e-deal (efficy corporate)** subdomain manages the integration between Apsis One and Efficy Corporate (E-deal) CRM systems. This includes installation of CRM connectors, bootstrap of unique keyspace identifiers per installation, routing of form submissions to create leads (Silhouettes), downloading and synchronizing contact data, and managing the distinct entity lifecycle (Persons vs. Silhouettes). The subdomain enforces strict separation between contact and lead/silhouette entities to prevent data model conflicts in consent exports, and ensures the CRM system remains the master of contact data while Apsis orchestrates lead-to-contact workflows.

---

## 2. Architecture Map

```
Installation Manager
├── Account Section Configuration (section discriminator)
├── Integration Configuration (CRM logical name, connection params)
└── Keyspace System
    ├── Main Keyspace (contact/person)
    ├── Entity-Specific Keyspaces (silhouette, lead)
    └── Discriminator Database (integrations:keyspaces:[8-char-hash]:[crm-logical-name])

Form Submission Flow
├── Form Campaign (sync to CRM option)
├── Event Listeners (start_viewed, submit events)
├── Form Submission Routing
├── E-deal CRM (form data in → entity type + ID out)
└── Profile Merging Engine (silhouette ↔ contact within keyspace)

Entity Synchronization
├── E-deal CRM (download Persons and Silhouettes)
├── Keyspace Lookup by Entity Type
├── Entity-Specific Keyspace Storage
└── Consent Export (contacts only; silhouettes excluded)

Base Installer (inheritance root)
└── Generic Connector
    └── E-deal/FCC Corporate Connector (inherits keyspace function, entity handling)

Cross-Integration Constraints
├── CRM is Master (no external keyspace merges)
├── Silhouette isolation (prevents silhouette inclusion in consent exports)
└── Keyspace persistence (no deletion on uninstall; supports re-engagement)
```

---

## 3. Module Reference

### Installation Manager
- **Purpose**: Bootstraps keyspaces and event listeners when E-deal CRM integration is installed on an account section.
- **Key files**: `Installation Manager` (main_install function)
- **Tech stack**: Apsis One Integrations platform
- **Data flow**: 
  - **In**: Account section configuration (section discriminator), Integration configuration (CRM logical name, connection params)
  - **Out**: Main keyspace created; entity-specific keyspaces created (contact, silhouette); discriminators stored in database
- **Business rules**:
  - Main keyspace created first for contact entity type
  - Loop through additional entity types (silhouette) and create dedicated keyspaces per entity
  - On reinstallation after uninstall, existing keyspace is **reused** (not recreated)
  - Keyspace is **never deleted** on integration uninstallation (intentional; preserves historical event data)
- **Configuration**:
  - Account section configuration must be active
  - CRM logical name must be defined (e.g., "E-deal", "efficy-corporate-12-1")
  - Entity types for E-deal: `{contact, silhouette}`
- **Integration points**: E-deal CRM (validates connection params during install)
- **Gotchas**:
  - If reinstalling, ensure section discriminator is identical or keyspace discriminator will differ
  - Uninstalling does NOT delete keyspace—reinstall reuses old keyspace and all historical data
  - Entity-specific keyspaces differ by CRM type; E-deal always includes silhouette keyspace
- **Tribal knowledge**:
  - Keyspace persistence on uninstall is deliberate: enables "win-back" customer scenarios where historical profile and event data must remain in Apsis for re-engagement campaigns
  - The 8-character hash is for uniqueness only; you cannot and should not attempt to reverse-resolve it to identify the source section

### Keyspace System
- **Purpose**: Uniquely identifies and manages contact/silhouette records across multiple E-deal installations and entity types within a single Apsis One account.
- **Key files**: Installation Manager (creates and manages); Keyspace Database (stores discriminator mappings)
- **Tech stack**: Apsis One Integrations platform (database-backed)
- **Data flow**:
  - **In**: CRM ID (returned from E-deal), entity type (contact or silhouette)
  - **Process**: Entity type → look up keyspace discriminator in database → retrieve keyspace ID → store/update entity in correct keyspace
  - **Out**: Entity data stored in entity-specific keyspace; consent exports routed from contact keyspace only
- **Business rules**:
  - Each E-deal installation gets a unique keyspace discriminator: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`
  - Hash (8 chars) ensures uniqueness across multiple E-deal installations on same account section
  - Keyspaces are never deleted; reused on reinstallation
  - Separate keyspaces for contact and silhouette prevent silhouette IDs from being exported back to E-deal in consent payloads
- **Configuration**:
  - Keyspace Discriminator Format: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`
  - Example: `integrations:keyspaces:a1b2c3d4:e-deal`
  - Hash algorithm: Section discriminator hashed (SHA-1 inferred; **⚠️ UNCONFIRMED**)
  - Hash length: 8 characters (first 8 chars of hash)
- **Integration points**: Database storage (discriminator lookup table); E-deal CRM (entity type responses)
- **Gotchas**:
  - If keyspace discriminator lookup fails, entity operations may fail silently
  - Section discriminator must be stable across installations; changes invalidate keyspace discriminator and break entity associations
  - 8-character hash collision unlikely but not explicitly mitigated; no reverse-resolution mechanism exists
- **Tribal knowledge**:
  - The 8-character hash length is intentional for readability when combined with CRM logical name; longer hashes add no practical value
  - Keyspace discriminators are stored in database during installation so they can be looked up later by entity type
  - Hash-based uniqueness is intentional design; system does not need to reverse-resolve the hash

### Discriminator Generation Function
- **Purpose**: Generates unique, deterministic keyspace discriminators from account section configuration and E-deal logical name.
- **Key files**: Base Installer (function defined here; inherited by E-deal Connector)
- **Tech stack**: Object-oriented inheritance pattern; hash algorithm (SHA-1 or equivalent)
- **Data flow**:
  - **In**: Account section configuration (section discriminator), Integration configuration (CRM logical name)
  - **Process**: 
    1. Extract section discriminator from account section
    2. Hash section discriminator (SHA-1 or equivalent)
    3. Extract first 8 characters of hash result
    4. Retrieve CRM logical name from integration configuration
    5. Construct discriminator: `integrations:keyspaces:[HASH_8_CHARS]:[CRM_LOGICAL_NAME]`
  - **Out**: Keyspace discriminator (string) and optional keyspace name/description
- **Business rules**:
  - Discriminator is deterministic: same section discriminator + CRM logical name always produces same discriminator
  - Discriminator is human-readable (includes CRM logical name for clarity)
  - First 8 characters of hash only (not variable length or full hash)
- **Configuration**:
  - Hash algorithm: **⚠️ Uncertain (SHA-1 or other?)**
  - Hash extraction: First 8 characters only
  - Format: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`
- **Integration points**: Base Installer (parent class providing function)
- **Gotchas**:
  - Hash algorithm must be consistent across installations; different algorithms produce different discriminators
  - Section discriminator changes invalidate existing keyspace discriminator
  - Case sensitivity of CRM logical name must be consistent
  - **⚠️ Hash algorithm implementation not confirmed in transcripts**
- **Tribal knowledge**:
  - Hash-based approach chosen to avoid reverse-resolution requirements; system does not need to identify source section from discriminator
  - 8-character hash balances collision resistance with readability

### Generic Connector (E-deal Implementation)
- **Purpose**: Abstract connector implementation for standard CRM system types; E-deal/Efficy Corporate inherits from this class.
- **Key files**: Not specified in transcript (likely in CRM connector module)
- **Tech stack**: Object-oriented inheritance; inherits from Base Installer
- **Data flow**:
  - **In**: Account section configuration, Integration configuration, Form submissions, CRM entity type responses
  - **Out**: Keyspace discriminator, Entity mappings to E-deal entities
- **Business rules**:
  - Inherits keyspace function from Base Installer
  - Maps Apsis Person/Contact entities to E-deal Persons
  - Handles Silhouette (lead) creation and lifecycle management
  - Enforces entity type separation (contact vs. silhouette)
- **Configuration**: Inherits from Base Installer; no E-deal-specific overrides documented
- **Integration points**: Base Installer (parent); E-deal CRM (external system)
- **Gotchas**:
  - E-deal returns 'silhouette' entity type for form submissions, not 'contact'; must store in silhouette keyspace
  - Silhouettes in contact keyspace would cause incorrect consent exports
- **Tribal knowledge**:
  - E-deal form submissions return 'silhouette' entity type (NOT 'contact'); this is critical distinction to prevent consent export bugs

### Base Installer
- **Purpose**: Blank boilerplate that fulfills all required functions for CRM connector implementations; root of inheritance hierarchy (including E-deal connector).
- **Key files**: Not specified in transcript
- **Tech stack**: Object-oriented programming with inheritance pattern
- **Data flow**: Receives connector configuration; distributes new functionality to all inheriting connectors (including E-deal Connector)
- **Business rules**:
  - All CRM connectors inherit from Base Installer
  - New functionality added to Base Installer is automatically available to all connector implementations
  - Keyspace function is defined here and inherited by all connectors (no connector-specific overrides documented)
- **Configuration**: Base class; configuration delegated to Account Section Configuration and Integration Configuration
- **Integration points**: All CRM connectors (E-deal, Microsoft Dynamics, Salesforce) inherit from this
- **Gotchas**:
  - Changes to Base Installer affect all connector implementations; must be carefully tested
- **Tribal knowledge**:
  - Base Installer pattern is critical to reducing code duplication across diverse CRM connectors
  - Adding a function once to Base Installer makes it available to all connectors without modification

### E-deal/FCC Corporate Connector
- **Purpose**: Integration adapter for E-deal (Efficy Corporate) CRM system; inherits from Base Installer.
- **Key files**: Not specified in transcript (likely in CRM connector module)
- **Tech stack**: Inherits from Base Installer or Generic Connector
- **Data flow**:
  - **In**: 
    - Account section configuration (for keyspace discriminator generation)
    - Form submissions with email, phone, fields
    - E-deal entity type responses (contact, silhouette)
    - E-deal CRM Person and Silhouette downloads
  - **Out**: 
    - Keyspace discriminators (to Installation Manager)
    - CRM IDs stored in entity-specific keyspaces
    - Profile updates sent to E-deal (for Persons only; not Silhouettes)
    - Consent exports (Persons only; Silhouettes excluded)
- **Business rules**:
  - Maps Apsis Person entities to E-deal Persons (main contact keyspace)
  - Creates Silhouettes via form submission (separate silhouette keyspace)
  - Main entities (Persons) downloaded from E-deal; Silhouettes created via form submission (not bulk downloaded)
  - Profile updates sent to E-deal for Persons only; Silhouettes do not receive profile updates
  - Silhouettes remain separate until explicit merge request from E-deal
  - E-deal is master of contact data; Apsis does not initiate merges
  - Consent exports include only Persons, not Silhouettes
- **Configuration**:
  - Entity types: `{contact, silhouette}`
  - Form minimum required fields: `email` (at minimum)
  - Main keyspace: created first for Person entity
  - Silhouette keyspace: created as entity-specific keyspace
- **Integration points**: 
  - E-deal CRM (form submissions, entity downloads, consent exports)
  - Installation Manager (keyspace bootstrap)
  - Profile Merging Engine (silhouette ↔ contact within keyspace)
  - Form Submission Routing
- **Gotchas**:
  - E-deal returns 'silhouette' entity type for form submissions, not 'contact'; storing in contact keyspace causes consent export bugs
  - Silhouettes must never be included in consent exports
  - Merging E-deal Persons with non-E-deal profiles (e.g., email keyspace) in Apsis will cause data inconsistency and will not propagate back to E-deal
  - Profile updates sent to E-deal will not be reflected back in Apsis; E-deal is master
- **Tribal knowledge**:
  - E-deal's distinction between Persons (main contacts) and Silhouettes (leads) is baked into the system; this is why separate keyspaces are necessary
  - Silhouette keyspace acts as holding area to prevent duplicate Person creation if same email submits form twice
  - When a Silhouette is promoted to a Person in E-deal, the merge must originate from E-deal; Apsis will receive updated entity type and perform internal merge

### Form Submission Routing Handler
- **Purpose**: Routes form submissions to correct E-deal keyspace and handles CRM responses (new records, existing records, entity type variations).
- **Key files**: Not specified in transcript
- **Tech stack**: Apsis One Integrations platform; event-driven
- **Data flow**:
  - **In**: 
    - Form submission event with user data (email, phone, form fields)
    - Profile key (unique Apsis profile identifier)
    - Form field values
  - **Process**: 
    1. Collect identifying data: email, phone, existing CRM ID, existing lead ID, form field values
    2. Construct payload: `{ profileKey, identifyingData: {email, phone, crmId, leadId}, fields: {...} }`
    3. Send to E-deal CRM
    4. Receive response: entity type (contact or silhouette) + ID
    5. Look up keyspace discriminator by entity type
    6. Store ID in corresponding entity-specific keyspace
    7. Perform merge if needed (silhouette ID + profile key)
  - **Out**: 
    - Form data sent to E-deal
    - CRM ID stored in appropriate keyspace
    - Merge performed if silhouette response
- **Business rules**:
  - Form sync requires explicit 'sync to CRM' option enabled
  - 'sync to CRM' option only visible if integration installed
  - Minimum required fields: email address
  - E-deal response determines which keyspace receives ID
  - Silhouette responses trigger merge with contact keyspace (via profile key)
- **Configuration**:
  - Form 'sync to CRM' visibility: Only visible/enabled if integration installed
  - Form submission payload structure: `{ profileKey, identifyingData: {email, phone, crmId, leadId}, fields: {...} }`
  - Minimum required fields: email
- **Integration points**: 
  - E-deal CRM (receives form submission, sends entity type + ID response)
  - Entity-Specific Keyspaces (stores CRM ID)
  - Profile Merging Engine (merge triggered if silhouette response)
  - Form Campaign Event Listeners
- **Gotchas**:
  - If integration not installed, 'sync to CRM' option will not appear and form will not route to E-deal
  - Entity type in E-deal response determines keyspace; if mismatched, data stored in wrong keyspace
  - E-deal form submissions always return 'silhouette' entity type (not 'contact'); must handle accordingly
  - Form submission only processed if integration is installed and active; under normal circumstances, no orphaned submissions
- **Tribal knowledge**:
  - Form sync 'sync to CRM' option visibility is a safety feature: prevents form submissions from being routed to inactive/uninstalled integrations
  - Profile key is added by Apsis to establish connection between form submission and Apsis profile

### Form Sync Event Listener System
- **Purpose**: Registers event listeners for form submission campaigns when 'sync to CRM' is enabled; manages form submission workflow.
- **Key files**: Not specified in transcript
- **Tech stack**: Apsis One Integrations event listener framework
- **Data flow**:
  - **In**: Form published with 'sync to CRM' option enabled
  - **Process**: 
    1. Detect 'sync to CRM' enabled on form
    2. Register event listeners for 'start_viewed' and 'submit' events
    3. On submit event: trigger Form Submission Routing Handler
  - **Out**: Event listeners active; form submissions routed to E-deal CRM
- **Business rules**:
  - Event listeners register for 'start_viewed' and 'submit' events
  - Listeners are removed on integration uninstall
  - Only active if integration is installed
- **Configuration**:
  - Event Listener Registration Trigger: Form published with 'sync to CRM' enabled
  - Events tracked: 'start_viewed', 'submit'
- **Integration points**: Form Campaign configuration; Form Submission Routing Handler; Installation Manager (listener removal on uninstall)
- **Gotchas**:
  - Listeners removed on integration uninstall; if integration reinstalled, listeners must be re-registered for active forms
  - If integration uninstalled but form still has 'sync to CRM' enabled (data persisted), listeners will not fire; form will not route to E-deal
- **Tribal knowledge**:
  - Event listeners are registration-based; reinstalling integration after uninstall requires re-registering listeners for existing forms with 'sync to CRM' enabled

### Entity-Specific Keyspaces
- **Purpose**: Separate storage containers for contact and silhouette entity types to prevent model conflicts and incorrect consent exports.
- **Key files**: Installation Manager (entity for loop); Keyspace System (storage)
- **Tech stack**: Apsis One Integrations platform (database-backed)
- **Data flow**:
  - **In**: Entity type from E-deal (contact or silhouette), CRM ID
  - **Process**: Entity type → look up discriminator → retrieve keyspace ID → store CRM ID in correct keyspace
  - **Out**: Contact IDs stored in contact keyspace; Silhouette IDs stored in silhouette keyspace
- **Business rules**:
  - Contact keyspace created first during installation
  - Silhouette keyspace created as entity-specific keyspace during installation (E-deal specific)
  - Silhouette IDs can **only** be stored in silhouette keyspace
  - Silhouette IDs in contact keyspace would trigger silhouette inclusion in consent exports (data corruption)
  - Contacts and silhouettes remain separate until explicit merge request from E-deal
- **Configuration**:
  - Entity types for E-deal: `{contact, silhouette}`
  - Discriminators per entity: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]:[entity-type]` (if needed for disambiguation)
  - Keyspace reuse: Existing keyspaces reused on reinstall (not deleted, not recreated)
- **Integration points**: Installation Manager (creation); Entity synchronization (storage/lookup); Consent export logic (silhouettes excluded)
- **Gotchas**:
  - Storing silhouette ID in contact keyspace is critical bug: triggers silhouette inclusion in consent exports
  - Different entity types require different discriminators; lookup must use correct entity type
  - If keyspace discriminator not found in database during lookup, operation may fail silently
- **Tribal knowledge**:
  - Silhouette keyspace acts as holding area to prevent duplicate contact creation if same person submits form twice with same email
  - Entity keyspace separation is necessary because E-deal's consent model requires contact-only exports; silhouettes should never be exported back to E-deal

### Profile Merging Engine
- **Purpose**: Merges silhouette records with contact records when indicated by E-deal response; operates only within E-deal entity keyspaces.
- **Key files**: Not specified in transcript
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Silhouette ID from form response, Profile key from form submission
  - **Process**: 
    1. Identify source keyspace: silhouette keyspace (with silhouette ID)
    2. Identify destination keyspace: contact/export keyspace (with profile key)
    3. Combine silhouette ID with contact profile key
    4. Store merged record
  - **Out**: Merged entity in contact keyspace; prevents duplicate creation on re-submission with same email
- **Business rules**:
  - Only merge within integration entity keyspaces (silhouette ↔ contact)
  - Never merge E-deal Persons with external keyspaces (e.g., email keyspace, other CRM keyspaces)
  - Merges initiated outside integration (e.g., in Apsis UI) risk data inconsistency and will not propagate back to E-deal
  - E-deal should be master of merge decisions; Apsis should not initiate merges
  - CRM is authoritative source; merges should originate from E-deal, not Apsis
- **Configuration**:
  - Merge constraint: Only within integration entity keyspaces
  - Prohibited: External merges (CRM-originated contacts with email keyspace or other profiles)
- **Integration points**: Entity-Specific Keyspaces (silhouette, contact); Form Submission Routing (triggered by silhouette response)
- **Gotchas**:
  - Merging E-deal Persons with non-E-deal profiles in Apsis causes data inconsistency; changes will not propagate back to E-deal
  - CRM should never receive merge requests from Apsis; CRM initiates merges via entity type changes in responses
  - If merge is incorrectly initiated in Apsis, CRM will have no knowledge of merge; data will diverge
- **Tribal knowledge**:
  - Silhouette keyspace holds temporary IDs until merge is triggered by silhouette response
  - Merge prevents duplicate contact creation if same email submits form twice; silhouette ID + profile key ties them together
  - CRM remains master; Apsis performs merge only based on CRM indication (entity type change in response)

### CRM Data Download Handler (Person Entities)
- **Purpose**: Synchronizes Person/Contact entities from E-deal CRM to Apsis.
- **Key files**: Not specified in transcript
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: E-deal entity list with entity type (contact, silhouette)
  - **Process**: 
    1. Initiate bulk download from E-deal
    2. Iterate through Person entities (exclude Silhouettes)
    3. For each Person: determine entity type, look up keyspace discriminator
    4. Store/update Person in contact keyspace using keyspace ID
  - **Out**: Person entities stored in contact keyspace; Silhouettes excluded
- **Business rules**:
  - Only Persons (contacts) are downloaded in bulk; Silhouettes are not
  - Silhouettes are created via form submission, not downloaded
  - Different CRM systems use different terminology (E-deal: "Person"; Microsoft Dynamics: "Contact"; Salesforce: "Contact" or "Lead")
  - Each Person stored with Person type discriminator in appropriate keyspace
- **Configuration**:
  - Entity download filter: Person type only
  - Entity type mapping: E-deal "Person" → contact keyspace
- **Integration points**: E-deal CRM (bulk export); Contact keyspace (storage); Keyspace System (lookup)
- **Gotchas**:
  - Silhouettes must not be included in Person bulk sync; different entity types require different handling
  - Different CRM systems use different terminology; connector must map to Apsis internal representation
  - If keyspace discriminator not found, sync may not complete
- **Tribal knowledge**:
  - Person download is bulk operation; Silhouette creation is event-driven (form submission)
  - E-deal Persons are contacts that can receive profile updates; this is why they are downloaded

### Consent Export Handler
- **Purpose**: Routes contact consent changes back to E-deal CRM; critically excludes Silhouettes.
- **Key files**: Not specified in transcript
- **Tech stack**: Apsis One Integrations platform
- **Data flow**:
  - **In**: Contact keyspace entities with consent changes
  - **Out**: Consent payload sent to E-deal (Persons only; Silhouettes excluded)
- **Business rules**:
  - Only contact keyspace entities included in consent exports
  - Silhouette keyspace entities explicitly excluded (critical: prevents sending silhouettes back to E-deal)
  - If silhouette ID mistakenly stored in contact keyspace, consent export will incorrectly include silhouette
- **Configuration**:
  - Export entity filter: Contact keyspace only
  - Silhouette exclusion: Enforced
- **Integration points**: Contact keyspace (read); E-deal CRM (consent export)
- **Gotchas**:
  - **CRITICAL**: Storing silhouette ID in contact keyspace causes consent export to include silhouette (data corruption)
  - If entity type handling in Form Submission Routing is incorrect, silhouettes may be stored in contact keyspace
- **Tribal knowledge**:
  - Silhouette keyspace separation exists primarily to prevent silhouette inclusion in consent exports
  - This is why E-deal form submissions returning 'silhouette' entity type must be handled specially

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **⚠️ Not documented in transcripts**
- E-deal CRM connection parameters (credentials, endpoints) stored in Integration Configuration
- No specific auth method specified (REST API credentials, OAuth, API keys assumed but unconfirmed)
- Installation Manager validates E-deal connection during install

### Error Handling & Logging
- **⚠️ Not documented in transcripts**
- Form Submission Routing may fail silently if keyspace discriminator lookup fails
- CRM response handling assumes entity type is always provided; missing type not specified
- Uninstallation does NOT return errors if keyspace is already deleted; reuse is silent

### Deployment & Lifecycle
- **Installation**: Installation Manager bootstraps keyspaces and event listeners
- **Uninstallation**: Integration uninstalled; event listeners removed; **keyspace preserved** (intentional)
- **Reinstallation**: Existing keyspace reused; event listeners re-registered; historical data retained
- **Multi-tenancy**: Unique keyspace discriminator per E-deal installation prevents entity collisions on same account section

### Data Consistency & Master Data
- **CRM is Master**: E-deal is authoritative source of contact data; Apsis does not override CRM data
- **Merges Originate in CRM**: Silhouette-to-contact conversion must be requested by E-deal, not initiated by Apsis
- **External Keyspace Merges Prohibited**: Merging E-deal Persons with email keyspace or other non-E-deal profiles in Apsis causes data divergence

### Form Submission Safety
- **Integration Presence Check**: 'sync to CRM' option only visible/available if integration installed
- **Keyspace Lookup Failure**: If discriminator lookup fails, entity operation may fail silently or route to wrong keyspace
- **Event Listener Registration**: Listeners must be active for form submissions to reach E-deal; uninstall removes listeners

---

## 5. Business Rules Reference

### Entity Type Management
| Rule | Context | Exception |
|------|---------|-----------|
| Main entities (Persons) are downloaded from E-deal CRM | Data synchronization | None |
| Lead entities (Silhouettes) are created only via form submission | Lead creation | Bulk download excludes Silhouettes |
| Apsis must track whether entity is Person or Silhouette | Entity lifecycle management | None |
| E-deal returns 'silhouette' for form submissions, not 'contact' | Form submission handling | E-deal-specific; other CRM systems may return 'lead' or 'contact' |
| Silhouettes are stored in separate keyspace from Persons | Entity isolation | None; critical for consent export correctness |

### Profile Updates & Synchronization
| Rule | Context | Exception |
|------|---------|-----------|
| Profile updates sent to E-deal for Persons only | CRM synchronization | Silhouettes do NOT receive profile updates |
| Silhouette does not receive profile updates after creation | Silhouette lifecycle | Silhouettes are temporary; conversion triggers Person sync |
| CRM is master of contact data | Data consistency | Apsis does not override CRM data |
| Changes synced from E-deal to Apsis are permanent (not deleted on uninstall) | Keyspace persistence | GDPR deletion requests override; otherwise no deletion |

### Keyspace & Multi-Tenancy
| Rule | Context | Exception |
|------|---------|-----------|
| Each E-deal installation gets unique keyspace | Multi-tenancy | Multiple E-deal instances on same account each get distinct discriminator |
| Keyspace discriminator is deterministic & reproducible | Installation repeatability | Must use same section discriminator for reproducible discriminator |
| Keyspaces are never deleted on integration uninstall | Historical data preservation | Reuse on reinstallation; no deletion except GDPR cleanup |
| Silhouettes cannot be added to contact keyspace | Data model integrity | Storing silhouette ID in contact keyspace causes consent export bugs |

### Form Submission & Lead Creation
| Rule | Context | Exception |
|------|---------|-----------|
| Form sync requires explicit 'sync to CRM' option enabled | Safety feature | Option hidden if integration not installed |
| Lead creation triggered by form submission with minimum data (email) | Lead workflow | Email required; other fields optional |
| When form submission creates lead, CRM responds with ID (not Person ID) | E-deal response handling | Silhouette ID returned; requires silhouette keyspace storage |
| Silhouette ID + profile key merged on response | Duplicate prevention | Prevents duplicate Person creation if same email submits twice |

### Merging & Data Integrity
| Rule | Context | Exception |
|------|---------|-----------|
| Profile merging only occurs within integration entity keyspaces | Data consistency | Merges: silhouette ↔ contact (within E-deal keyspace) |
| Never merge E-deal Persons with external profiles (email keyspace, etc.) | CRM data integrity | Merges in Apsis will not propagate back to E-deal; causes divergence |
| Merges must originate from CRM (E-deal), not Apsis | Master data principle | Apsis responds to merge requests from E-deal; does not initiate |

---

## 6. Known Issues & Workarounds

### ⚠️ Hash Algorithm Uncertainty
- **Issue**: Exact hash algorithm used for section discriminator hashing is uncertain (SHA-1 vs. other)
- **Impact**: Cannot validate discriminator generation without algorithm confirmation
- **Workaround**: Confirm and document exact hash algorithm in Discriminator Generation Function implementation
- **Severity**: Medium
- **Affected Component**: Keyspace System, Installation Manager

### Silhouette Entity Type Handling (E-deal Specific)
- **Issue**: E-deal form submissions return 'silhouette' entity type (not 'contact'); storing in contact keyspace causes silhouettes to be included in consent exports (data corruption)
- **Impact**: Silhouettes incorrectly exported back to E-deal; consent data inconsistency
- **Workaround**: Form Submission Routing Handler must check entity type response and route to silhouette keyspace for 'silhouette' responses
- **Severity**: Critical
- **Affected Component**: Form Submission Routing, Entity-Specific Keyspaces, Consent Export Handler

### Profile Update Flow from E-deal to Apsis
- **Issue**: Reverse synchronization flow (E-deal changes → Apsis updates) not detailed in transcripts
- **Impact**: Unknown how Person profile changes in E-deal are reflected back in Apsis
- **Workaround**: Refer to follow-up KT session or profile synchronization documentation
- **Severity**: Low
- **Affected Component**: CRM Data Download Handler, Person synchronization

### Account Section Configuration Structure
- **Issue**: Account section configuration structure and purpose not fully elaborated
- **Impact**: Cannot fully understand how section discriminator is extracted and validated
- **Workaround**: Refer to Account Configuration documentation or follow-up session
- **Severity**: Low
- **Affected Component**: Installation Manager, Keyspace System

### Keyspace Discriminator Database Lookup Failure
- **Issue**: If keyspace discriminator not found in database, Form Submission Routing or CRM Download may fail silently
- **Impact**: Form submissions or entity downloads route to wrong keyspace or fail to complete
- **Workaround**: Add validation in Installation Manager to verify discriminator is stored in database; add error handling in lookup operations
- **Severity**: Medium
- **Affected Component**: Installation Manager, Form Submission Routing, CRM Data Download Handler

### Lead-to-Person Conversion Workflow
- **Issue**: How E-deal indicates Silhouette → Person conversion (merge request) not fully specified
- **Impact**: Unknown if conversion triggers automatic merge in Apsis or requires explicit CRM response
- **Workaround**: Confirm conversion mechanism with E-deal integration team; verify merge trigger logic
- **Severity**: Low
- **Affected Component**: Profile Merging Engine, Form Submission Routing

### Silhouette/Lead ID Deletion Handling
- **Issue**: What happens to Silhouette ID if lead is deleted from E-deal? Is deletion communicated back to Apsis?
- **Impact**: Unknown if deleted Silhouettes are cleaned up in Apsis or remain orphaned
- **Workaround**: Confirm deletion workflow with E-deal integration team; implement cleanup if needed
- **Severity**: Low
- **Affected Component**: Consent Export Handler, Entity-Specific Keyspaces

### Section Discriminator Stability
- **Issue**: If section discriminator changes, existing keyspace discriminator is invalidated and entity associations break
- **Impact**: Changing account section configuration invalidates all E-deal entity associations in Apsis
- **Workaround**: Document that section discriminator must be stable; implement validation to prevent unintended changes
- **Severity**: Medium
- **Affected Component**: Installation Manager, Keyspace System

---

## 7. Glossary

| Term | Definition | E-deal Context |
|------|-----------|-----------------|
| **Person** | A fully-qualified customer or prospect record in E-deal CRM; has sufficient data to be a contact. These are main entities downloaded from E-deal and receive profile updates. | E-deal terminology: "Person"; equivalent to "Contact" in Microsoft Dynamics, "Contact" in Salesforce |
| **Silhouette** | A lead entity in E-deal CRM; represents prospective customer with minimal data (e.g., form submission data). Created when user submits Apsis form; does NOT receive profile updates. | E-deal-specific term; equivalent to "Lead" in other CRM systems |
| **Keyspace** | Unique namespace identifier generated per E-deal installation in Apsis. Ensures entities from E-deal do not collide with other CRM installations. Includes keyspace name, description, and discriminator. | Format: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]` |
| **Keyspace Discriminator** | Unique identifier for a keyspace, generated deterministically from account section configuration (hashed) and CRM logical name. | Format: `integrations:keyspaces:[8-char-hash]:[e-deal]` |
| **Section Discriminator** | Unique identifier within account configuration that is hashed to produce part of keyspace discriminator. Must be stable across E-deal connector installations. | Account-level; shared across all E-deal installations on same account section |
| **Entity-Specific Keyspace** | Keyspace dedicated to specific entity type (contact, silhouette). Prevents model conflicts and incorrect exports. | E-deal has contact and silhouette keyspaces |
| **Profile Key** | Unique identifier for an Apsis profile. Added to form submission payloads to identify profile on Apsis side. | Not same as CRM ID; links Apsis profile to E-deal entity |
| **CRM ID / Record ID** | Unique identifier for an entity (Person, Silhouette) in E-deal CRM. Returned in E-deal response to form submissions or downloads. | Stored in corresponding entity keyspace; links E-deal entity to Apsis profile |
| **Silhouette ID** | CRM ID returned by E-deal when Silhouette is created via form submission. | Must be stored in silhouette keyspace, NOT contact keyspace |
| **Form Sync** | Feature enabling form submissions to be automatically sent to E-deal CRM. Must be explicitly enabled per form; only visible if integration is installed. | 'sync to CRM' option in form configuration |
| **Event Listener** | Mechanism that registers for specific events (start_viewed, submit) on form campaigns when sync to CRM is enabled. Triggers E-deal synchronization. Removed on integration uninstall. | Removed on uninstall; must be re-registered on reinstall |
| **Merge Request** | Request from E-deal to merge related entity records (e.g., Silhouette → Person conversion). Within integration, triggers merge between silhouette and contact keyspaces. | E-deal indicates via entity type change in response |
| **Export Keyspace** | Keyspace containing profile key (used in merges to tie together Silhouette ID and profile key information). | Internal term; part of contact keyspace conceptually |
| **CRM is Master** | Design principle: E-deal is authoritative source of contact data. Apsis does not override E-deal data; merges originate from E-deal, not Apsis. | Enforced throughout integration flow |
| **Consent Export** | Process of sending contact consent changes back to E-deal CRM. Critically sensitive: Silhouette IDs must never be included. | Contact keyspace only; silhouette keyspace excluded |
| **Multi-Tenancy** | Support for multiple E-deal installations within single Apsis instance. Keyspaces enable this by providing unique entity namespaces per installation. | Unique discriminator per installation prevents collisions |
| **Installation Manager** | Bootstraps keyspaces and event listeners when E-deal integration is installed. Creates main and entity-specific keyspaces. | Executed once per E-deal installation per account section |
| **Keyspace Reuse** | On reinstallation of E-deal after uninstall, existing keyspace is reused (not deleted, not recreated). Historical data preserved. | Enables "win-back" scenarios; data never deleted unless GDPR cleanup requested |

---

## 8. Confidence Notes

### High Confidence
✅ Keyspace discriminator format and generation process
✅ Entity type separation (contact vs. silhouette) and isolation requirement
✅ Base Installer inheritance pattern and code reuse rationale
✅ Form submission routing to E-deal CRM
✅ Event listener registration and removal lifecycle
✅ Keyspace persistence on uninstall (intentional design)
✅ CRM is Master principle (no external keyspace merges)
✅ E-deal form submissions return 'silhouette' entity type
✅ Installation Manager bootstrap process
✅ Silhouettes not included in consent exports

### Medium Confidence
⚠️ Hash algorithm (assumed SHA-1; not explicitly confirmed)
⚠️ Hash length rationale (8 characters; choice not fully justified)
⚠️ Form submission payload structure (structure inferred; exact schema not specified)
⚠️ CRM response handling (entity type and ID expected; error cases not detailed)
⚠️ Keyspace database schema (discriminator mapping structure not specified)
⚠️ Reverse sync from E-deal to Apsis (flow not detailed)
⚠️ Lead-to-Person conversion mechanism (triggering condition not fully specified)

### Low Confidence / Unconfirmed
❓ Exact hash algorithm implementation (SHA-1 or other?)
❓ Full Account Section Configuration structure
❓ E-deal-specific API protocol details (REST, SOAP, or custom?)
❓ E-deal authentication method (API key, OAuth, credentials?)
❓ Silhouette deletion workflow (if deleted in E-deal, communicated to Apsis?)
❓ Keyspace database table structure (columns, indexes, constraints)
❓ Error handling and logging strategy (silent failures, logging detail level)
❓ Connector-specific overrides to Base Installer (if any exist for E-deal)

### Discrepancies & Conflicts
- **Keyspace Discriminator Format**: 
  - Session 1 specifies: `integration.keyspaces.[HASH_8_CHARS].[CRM_LOGICAL_NAME]`
  - Session 2 specifies: `integrations:keyspaces:[8-char-hash]:[CRM-logical-name]`
  - **Resolution**: Using Session 2 format (more recent, with colons instead of dots). Session 1 format may be outdated or alternative notation.

- **Hash Algorithm**:
  - Session 1 suggests: "SHA-1 or equivalent"
  - Session 2 does not confirm algorithm
  - **Resolution**: Marked as ⚠️ UNCONFIRMED; implementation must be verified

- **Silhouette Storage Context**:
  - Session 1: "Silhouettes created via form submission; not downloaded"
  - Session 2: "Silhouettes in E-deal form responses; stored in silhouette keyspace"
  - **Resolution**: Consistent; silhouettes are event-driven (form submission), not bulk download

### Outdated / Superseded Information
- Session 1 uses dot notation (`integration.keyspaces.[HASH].`) vs. Session 2 colon notation (`integrations:keyspaces:[HASH]:`); Session 2 is likely correct (more recent)

---

## 9. Tribal Knowledge (Non-Obvious Facts)

### Architectural Decisions
1. **Base Installer Inheritance Pattern**: Adding a function once to Base Installer automatically makes it available to all CRM connectors without modification. This eliminates duplication across Microsoft Dynamics, E-deal, Salesforce, and other connectors. Changes to Base Installer affect all connectors; careful testing required.

2. **Keyspace Persistence on Uninstall**: Keyspaces are **never deleted** on integration uninstallation. This is deliberate to preserve historical event data and enable "win-back" customer scenarios. If E-deal is uninstalled and reinstalled, all historical profiles and events remain in Apsis.

3. **Entity Keyspace Separation**: Separate keyspaces for silhouettes and persons exist primarily to prevent silhouettes from being included in consent exports. If a silhouette ID is mistakenly stored in the contact keyspace, consent export logic will incorrectly send the silhouette back to E-deal (data corruption).

4. **8-Character Hash Rationale**: The 8-character hash balances collision resistance with readability when combined with CRM logical name. Shorter hashes risk collisions; longer hashes add no practical value. The hash is for uniqueness only; system does not need reverse-resolution capability.

### Business Process Insights
5. **Form Sync Safety Feature**: The 'sync to CRM' option is hidden if the E-deal integration is not installed. This prevents form submissions from being routed to inactive/uninstalled integrations. Under normal circumstances, form submissions will never reach integrations that aren't active.

6. **Silhouette as Holding Area**: The silhouette keyspace acts as a temporary holding area. If the same person submits a form twice with the same email, the silhouette ID + profile key merge prevents duplicate Person creation in Apsis.

7. **Lead Creation Workflow**: E-deal form submissions return 'silhouette' entity type (not 'contact'). This is critical: silhouettes created from forms remain separate from persons until explicit merge request from E-deal. Sales team is notified for outreach; marketing automation can target silhouettes separately.

8. **CRM is Master Principle**: E-deal is the authoritative source of contact data. Apsis does not initiate merges or override CRM data. If a merge needs to happen, E-deal must indicate it via entity type change in response. Merging CRM-originated contacts with non-CRM profiles in Apsis will cause data divergence and inconsistency.

### Implementation Gotchas
9. **Section Discriminator Stability**: The section discriminator must be stable across E-deal installations. If it changes, the keyspace discriminator becomes invalid and entity associations break. This is a critical data preservation requirement.

10. **Hash Algorithm Consistency**: The hash algorithm used for section discriminator hashing must be consistent across installations. Different algorithms produce different discriminators. If algorithm changes, existing discriminators become invalid.

11. **Silhouette in Contact Keyspace Bug**: Storing a silhouette ID in the contact keyspace is a critical bug that causes consent exports to incorrectly include silhouettes. This violates E-deal's consent model and causes data sync errors.

12. **Entity Type Lookup Failure**: If a keyspace discriminator is not found in the database during lookup, entity operations may fail silently or route to the wrong keyspace. No error is raised; behavior depends on fallback logic (if any).

13. **Event Listener Removal on Uninstall**: Uninstalling E-deal removes event listeners for form campaigns. If the form still has 'sync to CRM' enabled (data persisted), listeners will not fire and the form will not route to E-deal on reinstall. Listeners must be re-registered.

### Design Principles
14. **No Reverse-Resolution of Hash**: The 8-character hash is intentionally non-reversible. System does not need to identify the source section from the discriminator. This simplifies the design and avoids collision vulnerability.

15. **Merges Only Within Integration**: Profile merges are only allowed within E-deal entity keyspaces (silhouette ↔ contact). Merges with external keyspaces (email keyspace, other CRM keyspaces) are prohibited because changes will not propagate back to E-deal and cause divergence.

16. **Historical Data Preservation**: Profiles synced from E-deal to Apsis remain in Apsis unless explicitly removed (GDPR deletion). This ensures historical profile and event data is available for re-engagement campaigns if E-deal integration is reinstalled.

---

## 10. Contact & Authority References

Based on KT sessions:
- **Erik** (referenced): Integration team member; attempting to simplify Tribe dynamic entity behavior; scheduled to follow-up on JoinCX custom keyspace request with Lukasz
- **Lukasz** (referenced): Team member; involved in investigation of per-customer keyspace customization request

For unresolved questions, escalate to integration architecture team or platform engineering.

---

## 11. Quick Reference: E-deal Integration Flow Diagram

```
┌─ Installation ─────────────────────────────────────────┐
│ Account Section Config (section discriminator)         │
│ Integration Config (CRM logical name, credentials)     │
│ ↓                                                       │
│ Installation Manager.main_install()                    │
│ ├─ Main Keyspace created                               │
│ │  └─ Discriminator: integrations:keyspaces:[HASH]:e-deal│
│ ├─ Entity-Specific Keyspace (silhouette)               │
│ │  └─ Discriminator: integrations:keyspaces:[HASH]:e-deal│
│ └─ Event Listeners registered for form campaigns       │
└──────────────────────────────────────────────────────────┘

┌─ Form Submission Flow ──────────────────────────────────┐
│ Form Submit Event (user fills form, clicks submit)     │
│ ↓                                                       │
│ Form Submission Routing Handler                         │
│ ├─ Collect: profileKey, email, phone, fields           │
│ ├─ Send to E-deal CRM                                   │
│ │  └─ E-deal responds: entityType (silhouette), ID     │
│ ├─ Look up keyspace discriminator by entity type       │
│ └─ Store ID in correct keyspace (silhouette keyspace)  │
│    ├─ Merge: silhouette ID + profile key               │
│    └─ Result: profile key + silhouette ID linked       │
└──────────────────────────────────────────────────────────┘

┌─ Person Download Flow ──────────────────────────────────┐
│ E-deal Sync Event                                       │
│ ├─ Initiate bulk download from E-deal                   │
│ ├─ E-deal returns: Persons (contacts)                   │
│ ├─ For each Person:                                     │
│ │  ├─ Entity type: contact                              │
│ │  ├─ Look up keyspace discriminator                    │
│ │  └─ Store in contact keyspace                         │
│ └─ Silhouettes excluded (not downloaded)                │
└──────────────────────────────────────────────────────────┘

┌─ Consent Export Flow ────────────────────────────────────┐
│ Consent Change Detected                                 │
│ ├─ Query contact keyspace (profiles with changes)      │
│ ├─ Exclude silhouette keyspace (CRITICAL)               │
│ ├─ Build export payload (Persons only)                  │
│ └─ Send to E-deal CRM                                   │
└──────────────────────────────────────────────────────────┘

┌─ Uninstallation Flow ────────────────────────────────────┐
│ Uninstall Triggered                                     │
│ ├─ Event Listeners removed                              │
│ └─ Keyspace PRESERVED (NOT deleted)                     │
│    └─ Historical data remains for re-engagement         │
└──────────────────────────────────────────────────────────┘

┌─ Reinstallation Flow ───────────────────────────────────┐
│ Installation Triggered (after prior uninstall)          │
│ ├─ Installation Manager checks: is keyspace existing?   │
│ ├─ Yes → Reuse existing keyspace                        │
│ │        Historical data + events restored              │
│ └─ Event Listeners re-registered                        │
└──────────────────────────────────────────────────────────┘
```

---

## 12. Key Takeaways for AI Coding Agent

**When implementing or modifying the E-deal integration:**

1. ✅ **Always check entity type** before storing IDs; route silhouettes to silhouette keyspace, persons to contact keyspace
2. ✅ **Never store silhouette IDs in contact keyspace**—this breaks consent exports
3. ✅ **Never merge E-deal persons with external keyspaces** (email, other CRMs)—changes won't propagate back to E-deal
4. ✅ **Preserve keyspaces on uninstall**—reuse on reinstall to maintain historical data
5. ✅ **Verify section discriminator is stable**—changes invalidate keyspace discriminators
6. ✅ **Check integration installed before routing form submissions**—'sync to CRM' option hidden if not installed
7. ✅ **E-deal is master of contact data**—Apsis responds to E-deal signals, doesn't initiate changes
8. ⚠️ **Confirm hash algorithm** (SHA-1?) and implementation before deployment
9. ⚠️ **Add error handling** for keyspace discriminator lookup failures
10. ⚠️ **Test silhouette-to-person conversion** workflow to confirm merge triggering mechanism

---

**Knowledge Base Generated**: 2026-03-25
**Last Updated**: From KT Session 2 (Keyspaces in Integrations part 2)
**Coverage**: e-deal (efficy corporate) subdomain within Apsis One Integrations
**Token Efficiency**: Deduplicated across 2 KT sessions; conflicts resolved; gaps flagged with ⚠️