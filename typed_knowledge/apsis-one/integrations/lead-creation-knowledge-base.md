---
title: Apsis One Integrations — lead creation Knowledge Base
subdomain: lead creation
generated: 2026-03-25T12:40:46.850Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (2 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — lead creation Knowledge Base](#apsis-one-integrations-lead-creation-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Form Tool](#form-tool)
    - [Audience Subscription Worker](#audience-subscription-worker)
    - [Kafka Queue (Batching)](#kafka-queue-batching)
    - [Outbound Worker](#outbound-worker)
    - [Merge Worker](#merge-worker)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Mappings Manager Service](#mappings-manager-service)
    - [Profile Identification System](#profile-identification-system)
    - [Key Space Manager](#key-space-manager)
    - [Base Installer](#base-installer)
    - [Generic Connector](#generic-connector)
    - [CRM Connectors (Microsoft Dynamics, E-deal/FCC Corporate, Salesforce)](#crm-connectors-microsoft-dynamics-e-dealfcc-corporate-salesforce)
    - [Keyspace Function](#keyspace-function)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling](#error-handling)
    - [Logging & Observability](#logging-observability)
    - [Deployment & Configuration Management](#deployment-configuration-management)
    - [Data Consistency & Transactions](#data-consistency-transactions)
    - [Multi-Tenancy & Isolation](#multi-tenancy-isolation)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Architecture & Data Ownership](#architecture-data-ownership)
    - [Form Syncing & Submissions](#form-syncing-submissions)
    - [Profile Management & Key Spaces](#profile-management-key-spaces)
    - [Field Mapping & Data Protection](#field-mapping-data-protection)
    - [Consent & Subscription Handling](#consent-subscription-handling)
    - [Lead Entities & Entity Types](#lead-entities-entity-types)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
  - [7. Glossary](#7-glossary)
    - [Core Concepts](#core-concepts)
    - [Integration Components](#integration-components)
    - [Data Flow Terms](#data-flow-terms)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence ✓](#high-confidence-)
    - [Medium Confidence ⚠️](#medium-confidence-)
    - [Low Confidence / Gaps ⚠️](#low-confidence-gaps-)
    - [Conflicting / Unresolved Information](#conflicting-unresolved-information)
    - [Documentation Gaps Requiring Follow-up](#documentation-gaps-requiring-follow-up)
  - [Unresolved Questions (Prioritized by Impact)](#unresolved-questions-prioritized-by-impact)
    - [Critical (Blocking Implementation)](#critical-blocking-implementation)
    - [High (Impacts Data Integrity)](#high-impacts-data-integrity)
    - [Medium (Affects UX / Reliability)](#medium-affects-ux-reliability)
    - [Low (Documentation / Nice-to-Have)](#low-documentation-nice-to-have)
  - [Key Decisions Requiring Confirmation / Clarification](#key-decisions-requiring-confirmation-clarification)
  - [Architectural Patterns & Design Decisions](#architectural-patterns-design-decisions)
    - [Inheritance for Code Reuse](#inheritance-for-code-reuse)
    - [Deterministic Keyspace Generation](#deterministic-keyspace-generation)
    - [Unidirectional Attribute Sync (with Consent Exception)](#unidirectional-attribute-sync-with-consent-exception)
    - [Event-Driven Form Submission Processing](#event-driven-form-submission-processing)
    - [Lead Entity Distinction](#lead-entity-distinction)

---

# Apsis One Integrations — lead creation Knowledge Base

## 1. Subdomain Overview

The **lead creation** subdomain manages the workflow of capturing prospective customer data through Apsis forms, transmitting form submissions as events to connected CRM systems, and orchestrating the creation of lead entities (Silhouettes, Leads) in those systems. It encompasses form syncing configuration, form submission event processing, profile identification and merging, and the distinction between main entities (Persons/Contacts) and lead entities (Silhouettes/Leads) that do not receive profile updates. This subdomain bridges Apsis' form tools with external CRM systems via an integration pipeline that respects the CRM as the authoritative source of contact data.

---

## 2. Architecture Map

**Core flow:**
- Form submission → Audience Service event → Audience Subscription Worker → Kafka Queue (batching) → Outbound Worker → CRM API
- CRM response (matched/new contact) → Merge Worker → Key Space consolidation
- Form syncing setup: Form Tool registers event listeners via Integration Service → CRM connector instantiated from Base Installer

**Multi-tenant keyspace architecture:**
- Account section configuration + CRM logical name → Keyspace function (deterministic hash) → unique discriminator per CRM installation
- Email/SMS key spaces hold public form profiles; CRM-specific key spaces (e.g., "FSC Enterprise 12.1") hold CRM-synced entities
- Profile merge consolidates email key space with CRM key space after form submission matches/creates CRM contact

**Entity types:**
- **Person** (Contact in Dynamics, Person in E-deal): downloaded from CRM, receives profile updates
- **Silhouette/Lead** (in E-deal/Salesforce): created only via form submission, does NOT receive profile updates

**Inheritance hierarchy:**
- Base Installer → Generic Connector → CRM-specific connectors (Dynamics, E-deal, Salesforce)

**Data sources:**
- Form Tool, Audience Service (inbound); CRM Systems (bidirectional); Mappings Manager Service (configuration)

---

## 3. Module Reference

### Form Tool
- **Purpose:** Creates, manages, and syncs forms to CRM systems; generates pre-filled form links and processes form submissions.
- **Key files:** Form Designer, Form Event Listeners, Pre-filled Form Link Generation, Form Submission Processing
- **Tech stack:** Form builder; integration with Apsis One platform
- **Data flow:**
  - Inbound: Form creation request, sync option selection (CRM target)
  - Outbound: Event to Audience Service (form submission), event to Integration Service (sync registration), query to Mappings Manager Service (field mapping)
- **Business rules:**
  - CRM sync option registers event listeners (open, submitted, started, viewed for form tools; all events for form event tools)
  - Mapped fields should block overwrites when profile has CRM ID (prevents later overwrite by Delta Sync)
  - CRM ID field must never be editable in form designer when integration exists
  - Enrichment attributes (unmapped fields) added via forms remain local to Apsis only
  - Email field is locked (read-only) in pre-filled forms; SMS-only profiles need verification of lock status
- **Configuration:**
  - Feature flag: `pre-filled-forms-enabled` (controls visibility of pre-filled form option)
  - Form field lock configuration: email locked in pre-filled forms; CRM ID hidden if integration active
  - Mapped field override protection: query Mappings Manager for field protection rules before accepting form submission updates
- **Integration points:**
  - Audience Service (form submission events)
  - Integration Service (form sync registration)
  - Mappings Manager Service (field mapping queries)
  - CRM Systems (form sync, pre-fill data)
- **Gotchas:**
  - ⚠️ If no identification data (no CRM ID, email, or SMS), submission is discarded silently
  - ⚠️ Form submission may overwrite mapped CRM attributes, which are then reverted by Delta Sync, confusing users
  - ⚠️ CRM ID field may be selectable/editable in form designer when integration is active; should be hidden entirely
  - ⚠️ Pre-filled form consent checkboxes are NOT pre-populated with profile's current consent status (bug/feature gap)
  - ⚠️ Pre-filled form links must be locked to identified profile to prevent unauthorized access
- **Tribal knowledge:**
  - Pre-filled forms add significant complexity to profile resolution because they send forms to existing profiles (possibly in CRM key space only); if wrong key space is queried, duplicate profiles are created
  - Form tool must query Mappings Manager dynamically per form submission, not rely on hard-coded field names, to determine field protection rules

### Audience Subscription Worker
- **Purpose:** Receives form submission events from Audience Service, validates identification data, converts events to integration format, and queues for CRM transmission.
- **Key files:** Audience Subscription Worker Queue, Event Format Converter
- **Tech stack:** Message queue consumer; integration event conversion
- **Data flow:**
  - Inbound: Event from Audience Service (form submission with user data)
  - Processing: Extract profile identification (CRM ID, email, phone), activity metadata (activity ID, timestamp, event type, profile key); validate at least one identification field present
  - Outbound: Event to Kafka Queue in integration format
- **Business rules:**
  - Form submission validation: requires at least one of CRM ID, email, or phone
  - Form submissions to CRM send only event notifications; do not send attribute data
  - Event format includes: activity ID, event type, profile fields (CRM ID, email, phone), timestamp
- **Configuration:**
  - Identification data validation: at least one required; if none present, submission discarded as "useless"
- **Integration points:**
  - Audience Service (inbound events)
  - Kafka Queue (outbound batching)
  - Profile identification system
- **Gotchas:**
  - ⚠️ If no identification data exists, submission is discarded silently with no error message to user
  - ⚠️ CRM ID is optional for public forms but required context for later merge operation

### Kafka Queue (Batching)
- **Purpose:** Aggregates individual form submission events for batch processing to reduce API calls to CRM.
- **Key files:** Form event batching queue
- **Tech stack:** Kafka message broker
- **Data flow:**
  - Inbound: Individual form events from Audience Subscription Worker
  - Processing: Batching worker aggregates events by batch size or time window
  - Outbound: Batches sent to Outbound Worker
- **Business rules:**
  - Events are batched for efficiency before transmission to CRM
- **Configuration:**
  - Batch size and time window (not specified in documentation; implies configurable)
- **Integration points:**
  - Audience Subscription Worker (inbound)
  - Outbound Worker (outbound)
- **Gotchas:**
  - Rate limiting and batch configuration unclear; may cause batching delays

### Outbound Worker
- **Purpose:** Sends batched form submission events to CRM system via HTTP requests and processes responses.
- **Key files:** Outbound integration requests, CRM API integration
- **Tech stack:** HTTP client; CRM API integration
- **Data flow:**
  - Inbound: Batch of form events from Kafka Queue
  - Processing: Format batch into HTTP POST request to CRM API endpoint
  - Outbound: HTTP POST to CRM; await response with matched/new contact ID or empty response
  - Response flow: Response sent to Merge Worker for key space consolidation
- **Business rules:**
  - CRM is authoritative; form event is notification only, not data payload
  - CRM response has three outcomes: `matched_record` (existing contact found), `new_record` (new contact created), or empty (no action)
- **Configuration:**
  - CRM API endpoint URL (per integration configuration)
  - Request format: HTTP POST with batch payload including activity ID, event type, profile identification, timestamp
- **Integration points:**
  - Kafka Queue (inbound batches)
  - CRM Systems (outbound HTTP API calls)
  - Merge Worker (response processing)
- **Gotchas:**
  - ⚠️ CRM response format varies by CRM system; parser must handle all three response types correctly
  - ⚠️ If Outbound Worker fails, batch is typically retried; ensure idempotency

### Merge Worker
- **Purpose:** Processes CRM responses to form submissions; consolidates profiles across key spaces when CRM returns contact ID.
- **Key files:** Merge worker queue, Key space merge logic
- **Tech stack:** Profile merge logic; key space management
- **Data flow:**
  - Inbound: CRM response with matched/new contact ID from Outbound Worker
  - Processing: Check if profile has CRM ID; if not, add from CRM response; initiate merge between email/SMS key space and CRM-specific key space
  - Outbound: Consolidated profile (same ID in both key spaces) stored in Apsis
- **Business rules:**
  - When form submission results in CRM contact match, merge must occur between email/SMS key space and CRM key space
  - Merge must be idempotent (safe to retry)
  - Original profile ID should be preserved; CRM ID added as attribute to same profile
  - Profile synced from CRM exists only in the CRM-specific key space (not email/SMS key space) unless explicitly merged
  - If merge fails or doesn't occur, duplicate profiles will be created in Apsis (one per key space)
- **Configuration:**
  - Merge logic configuration (not detailed)
- **Integration points:**
  - Outbound Worker (inbound CRM responses)
  - Profile Management (key space operations)
  - CRM Key Spaces (FSC Enterprise 12.1, Dynamics, E-deal, etc.)
- **Gotchas:**
  - ⚠️ If merge doesn't occur, two profiles will exist in Apsis (one per key space); very difficult to fix post-creation
  - ⚠️ Empty CRM responses do NOT trigger merges; profiles created in email key space stay there if not matched/created in CRM
  - ⚠️ Form submission may succeed but create duplicate silently (no error message)

### Delta Sync Worker
- **Purpose:** Continuously syncs contact attribute changes from CRM to Apsis; updates profile data based on CRM changes and overwrites locally modified attributes.
- **Key files:** Delta sync scheduling, CRM API polling/webhook integration
- **Tech stack:** Scheduled sync; CRM API polling or webhook
- **Data flow:**
  - Inbound: Scheduled wake; query CRM API for contact changes since last sync timestamp
  - Processing: For each changed contact, extract CRM ID and changed attributes; look up profile using CRM ID in CRM-specific key space; update profile attributes with CRM values (overwrites local values)
  - Outbound: Updated profile attributes; updated sync timestamp; logged conflicts/failures
- **Business rules:**
  - CRM is master of contact attributes; Apsis never sends attribute updates to CRM from form submissions
  - Mapped field overwrites by form submission will be reversed on next Delta Sync from CRM (by design)
  - Delta Sync ALWAYS overwrites local Apsis attribute values with CRM values (no conflict resolution)
  - Consent (subscription preferences) is bidirectional and synced to CRM if subscription is in consent mapping (exception to "CRM master" rule)
  - Delta Sync must use CRM ID key space to locate profiles; if profile in email key space only, Delta Sync won't find it
  - Sync must be idempotent (safe to run multiple times)
- **Configuration:**
  - Sync schedule: frequency configurable (implied daily or hourly; no exact value documented)
  - Attribute overwrite behavior: always overwrites (no user override documented)
- **Integration points:**
  - CRM Systems (inbound attribute changes via API polling or webhook)
  - Profile Management (attribute updates)
  - Key Space Management (uses CRM ID to locate profiles)
- **Gotchas:**
  - ⚠️ Delta Sync OVERWRITES locally modified attributes with CRM values; if form submission modified mapped field, Delta Sync will revert it
  - ⚠️ If form submission modified an attribute and Delta Sync runs immediately after, form change is silently overwritten with no conflict resolution
  - ⚠️ Delta Sync is unaware of recent form submissions to same profiles; no coordination mechanism
  - ⚠️ If profile has CRM ID in email key space only (not merged to CRM key space), Delta Sync won't find it and changes won't be synced
  - ⚠️ Must be idempotent; retry logic must not cause duplicate updates

### Mappings Manager Service
- **Purpose:** Provides mapping configuration between Apsis fields and CRM fields; enables Form Tool to determine which fields are mapped vs. enrichment fields.
- **Key files:** Mappings Manager Service API, Field mapping definitions
- **Tech stack:** Configuration service API
- **Data flow:**
  - Inbound: Query with account + section + integration ID
  - Outbound: List of mapped fields and their CRM equivalents (e.g., {apsis_field: 'first_name', crm_field: 'FirstName'})
- **Business rules:**
  - Mapped fields have corresponding fields in the CRM system (configured in Mappings Manager)
  - Enrichment attributes (unmapped fields) added via forms exist only in Apsis and are not synced to CRM
  - Consent mapping is configured per subscription in Mappings Manager; if subscription not in mapping, consent changes are ignored by integration
  - Form tool should query Mappings Manager to determine field update permissions before processing form submissions
- **Configuration:**
  - Field mapping definitions: account + section + integration ID → list of {apsis_field, crm_field, field_type, read_only_flag, etc.}
  - Consent mapping: subscription → CRM subscription field
- **Integration points:**
  - Form Tool (field mapping queries)
  - Integration Pipeline (uses mappings for field protection and consent routing)
- **Gotchas:**
  - ⚠️ Mappings Manager query failure should cause form submission to be rejected (fail safe behavior)
  - ⚠️ Field name matching must be exact (case-sensitive) between form fields and mapped field names
  - ⚠️ Unclear how pre-filled form tool currently accesses mappings; may need API integration
  - ⚠️ Consent fields are special case: even if mapped, should allow overwrites for bidirectional sync

### Profile Identification System
- **Purpose:** Locates profiles in correct key space (email, SMS, CRM ID); handles profile resolution for form submissions and pre-filled forms.
- **Key files:** Profile resolution algorithm, Key space routing
- **Tech stack:** Key space lookup; profile resolution logic
- **Data flow:**
  - Inbound: Profile identifier (email, phone, CRM ID, or profile key) from form submission or pre-filled form link
  - Processing: Query correct key space for identifier; if found, return profile ID; if not found, create new profile in appropriate key space
  - Outbound: Profile ID or new profile record
- **Business rules:**
  - Form submission must validate identification data exists (CRM ID, email, or phone)
  - Profile must be resolvable to correct key space to avoid creating duplicates
  - Profiles synced from CRM exist only in CRM-specific key space (not email/SMS key space) unless explicitly merged
  - Pre-filled form profile resolution must correctly identify profile to avoid creating duplicate in email key space
- **Configuration:**
  - Key space routing logic (account section + integration ID → CRM key space)
- **Integration points:**
  - Form Tool (pre-filled form link validation; form submission profile lookup)
  - Audience Subscription Worker (profile lookup for form submission)
  - Key Space Manager (queries)
- **Gotchas:**
  - ⚠️ Which key space does form tool use to locate profile when pre-filled form is submitted? Critical for preventing duplicates; unresolved question
  - ⚠️ If form tool doesn't query CRM key space, duplicate created in email key space if profile exists only in CRM key space
  - ⚠️ If email belongs to profile in CRM key space, must detect and merge key spaces; unclear if currently implemented

### Key Space Manager
- **Purpose:** Manages Apsis key spaces (email, SMS, CRM-specific key spaces); handles profile merges between key spaces; bootstraps CRM-specific key spaces on integration installation.
- **Key files:** Key space definitions, Profile merge operations, CRM key space bootstrapping
- **Tech stack:** Key space registry; merge logic
- **Data flow:**
  - Inbound: New CRM integration installed (triggers bootstrap); merge requests from Merge Worker
  - Processing: Create CRM-specific key space (e.g., 'FSC Enterprise 12.1'); execute merge between key spaces
  - Outbound: Bootstrapped key space definition; consolidated profile in merged key spaces
- **Business rules:**
  - Each CRM installation gets unique key space to avoid entity collisions in multi-tenant environments
  - Key space discriminator generated deterministically from section discriminator + CRM logical name (ensures reproducibility)
  - Keyspace name format: `integration.keyspaces.[8-char-hash].[CRM_LOGICAL_NAME]`
  - Merge operation consolidates profiles so both key space lookups resolve to same profile ID
  - Email and SMS key spaces are standard; CRM-specific key spaces created per integration
- **Configuration:**
  - Key space definitions (generated automatically on CRM integration install)
  - Key space discriminator function: hash(section_discriminator) + CRM_logical_name
  - Hash algorithm: SHA-1 (or equivalent; exact algorithm unconfirmed ⚠️)
  - Hash length: first 8 characters
- **Integration points:**
  - Merge Worker (initiates merges)
  - Profile Management (enforces key space isolation)
  - CRM Integration (bootstraps CRM-specific key spaces)
  - Base Installer (Keyspace function)
- **Gotchas:**
  - ⚠️ Section discriminator changes invalidate existing key space discriminator; breaks profile associations
  - ⚠️ Hash algorithm must be consistent across installations; different algorithm produces different discriminators
  - ⚠️ Hash must always use first 8 characters (not variable length)
  - ⚠️ Logical name case sensitivity must be consistent
  - ⚠️ Exact hash algorithm (SHA-1?) unconfirmed; needs documentation

### Base Installer
- **Purpose:** Blank boilerplate class that fulfills all required functions for CRM connector implementations; serves as root of inheritance hierarchy to eliminate code duplication.
- **Key files:** Base Installer implementation (exact file path not documented)
- **Tech stack:** Object-oriented programming with inheritance pattern
- **Data flow:**
  - Inbound: CRM integration configuration
  - Outbound: Inherited functions propagated to all CRM-specific connectors
- **Business rules:**
  - New functionality added to Base Installer is inherited by all connector implementations (Dynamics, E-deal, Salesforce)
  - Keyspace function defined on Base Installer; inherited by all connectors
  - CRM connector installation triggers Base Installer instantiation for CRM type
- **Configuration:**
  - Keyspace function configuration (inherited by all connectors)
  - CRM logical name configuration
- **Integration points:**
  - Generic Connector (inherits from Base Installer)
  - All CRM-specific connectors (inherit from Base Installer)
  - Keyspace function (defined here)
- **Gotchas:**
  - ⚠️ Base Installer is abstract; not directly instantiated
  - ⚠️ Changes to Base Installer affect all downstream connectors; requires careful testing

### Generic Connector
- **Purpose:** Abstract connector implementation for standard CRM system types; provides default behavior inherited by specialized connectors.
- **Key files:** Generic Connector implementation (exact file path not documented)
- **Tech stack:** Inherits from Base Installer
- **Data flow:**
  - Inbound: CRM configuration
  - Processing: Delegates to CRM system; uses inherited Keyspace function
  - Outbound: Connector ready for form submissions and entity synchronization
- **Business rules:**
  - Inherits all Base Installer functions; can specialize as needed
  - Used for CRM systems that follow common integration pattern
- **Integration points:**
  - Base Installer (parent)
  - CRM systems (specialized implementations)
- **Gotchas:**
  - Not all CRM connectors are "generic"; some may inherit directly from Base Installer and override functions

### CRM Connectors (Microsoft Dynamics, E-deal/FCC Corporate, Salesforce)
- **Purpose:** Integration adapters for specific CRM systems; inherit from Base Installer or Generic Connector; handle CRM-specific entity mapping and API integration.
- **Key files:** Connector-specific implementation (exact paths not documented)
- **Tech stack:** CRM-specific API integration; inherit from Base Installer/Generic Connector
- **Data flow:**
  - Inbound: Form submissions, CRM configuration, CRM-specific entity data
  - Processing: Map Apsis entities (Person, Silhouette/Lead) to CRM entities; handle CRM-specific API calls; parse CRM responses
  - Outbound: Lead/contact creation responses with ID; person entity downloads
- **Business rules:**
  - **Microsoft Dynamics:** Maps Apsis Person/Contact entities to Dynamics contacts; handles lead creation responses
  - **E-deal:** Maps Apsis Person entities to E-deal Persons; handles Silhouette (lead) creation and lifecycle; supports Silhouette ID responses
  - **Salesforce:** Handles Salesforce Lead and Contact entity mapping; supports lead creation from form submissions
  - Main entities (Persons) are downloaded from CRM systems; lead entities (Silhouettes) are not
  - Profile updates are sent to CRM for main entities only; lead entities do not receive profile updates
  - Lead entities are created only when user submits Apsis form with minimum required data (email address)
- **Configuration:**
  - CRM system credentials (not detailed)
  - CRM logical name (appended to key space discriminator)
  - Entity mapping configuration (Person/Contact, Silhouette/Lead)
  - Minimum required fields for lead creation (email address)
- **Integration points:**
  - Base Installer (inherits functions)
  - CRM Systems (entity download, form submission events, profile updates, Delta Sync)
  - Keyspace function (inherited; generates CRM-specific key space discriminator)
- **Gotchas:**
  - ⚠️ Different CRM systems use different terminology (Contact vs. Person vs. Lead); connector must abstract this
  - ⚠️ Minimum required fields for lead creation may vary by CRM system (email documented; others unclear)
  - ⚠️ Profile update flow from CRM back to Apsis not detailed; refer to follow-up session
  - ⚠️ Lead-to-person conversion workflow unclear; needs documentation

### Keyspace Function
- **Purpose:** Generates unique, deterministic keyspace identifiers and discriminators for each CRM installation based on account configuration.
- **Key files:** Function defined on Base Installer (inherited by all connectors)
- **Tech stack:** Hash algorithm function (SHA-1 or equivalent)
- **Data flow:**
  - Inbound: Account section configuration (with section discriminator) + CRM integration configuration (with CRM logical name)
  - Processing: Extract section discriminator; hash using hash algorithm; take first 8 characters; combine with CRM logical name
  - Outbound: Keyspace name, description, discriminator
- **Business rules:**
  - Keyspace discriminator is unique per CRM installation
  - Discriminator is reproducible; same inputs always produce same output
  - Discriminator format: `integration.keyspaces.[8-char-hash].[CRM_LOGICAL_NAME]`
  - Hash algorithm must be consistent across installations
  - Section discriminator must be stable; changes invalidate keyspace discriminator
- **Configuration:**
  - Hash algorithm: SHA-1 (or equivalent; unconfirmed ⚠️)
  - Hash length: first 8 characters
  - CRM logical name appended (e.g., 'Enterprise', 'Salesforce', 'E-deal', 'Dynamics')
- **Integration points:**
  - Account section configuration (input)
  - Integration configuration (input)
  - Base Installer (where function is defined)
  - All CRM connectors (inherit this function)
- **Gotchas:**
  - ⚠️ Hash algorithm must be documented and consistent; currently unconfirmed
  - ⚠️ Section discriminator changes break keyspace associations; must be stable
  - ⚠️ Hash must always use first 8 characters (not variable); collision resistance vs. readability tradeoff
  - ⚠️ CRM logical name case sensitivity must be consistent
  - ⚠️ "Account section configuration" structure not fully elaborated; needs follow-up

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- CRM API authentication: exact mechanism not detailed; implies API keys, OAuth, or similar per CRM system
- Internal service-to-service authentication: Form Tool → Integration Service, Audience Service → Audience Subscription Worker, Form Tool → Mappings Manager Service (not detailed)
- Pre-filled form links: locked to profile to prevent unauthorized access; email field confirmed locked; SMS-only field lock status unverified

### Error Handling
- Form submission with no identification data: discarded silently (no error message to user)
- Merge Worker failure: typically retried asynchronously; ensure idempotency
- Mappings Manager query failure: should cause form submission rejection (fail safe)
- Delta Sync conflicts: logged but no conflict resolution (CRM always wins)
- Pre-filled form field lock: no documented error when user attempts to modify locked field

### Logging & Observability
- Delta Sync Worker: logs conflicts and failures (no detail on logging mechanism)
- Merge Worker: no explicit logging requirements documented
- Outbound Worker: assumes logging of HTTP requests/responses to CRM (standard practice)
- Form submission tracking: unclear how field source is tracked (CRM vs. form vs. enrichment)

### Deployment & Configuration Management
- Feature flags: `pre-filled-forms-enabled` (feature flag controls visibility; status hidden/restricted)
- Environment variables: not documented; assume per-environment CRM credentials
- Integration configuration: account section + CRM system type + CRM credentials (no secrets management detailed)
- Keyspace configuration: auto-generated on CRM integration install (no manual configuration required)

### Data Consistency & Transactions
- Profile merge: must be idempotent (safe to retry); no explicit transaction guarantees documented
- Delta Sync overwrite: no conflict resolution; CRM always wins by design
- Form submission batching: assume ACID guarantees via Kafka (standard practice)
- Consent bidirectional sync: flow is form → Apsis → integration → CRM → Delta Sync back to Apsis (no conflict resolution if CRM and Apsis have different values)

### Multi-Tenancy & Isolation
- Key spaces provide entity namespace isolation per CRM installation
- Account section discriminator hashed to ensure unique key spaces even if same section name used across accounts
- CRM logical name appended to discriminator for human readability
- No documented isolation between different Apsis accounts/instances

---

## 5. Business Rules Reference

### Architecture & Data Ownership

| Rule | Context | Exceptions |
|------|---------|-----------|
| CRM is master of contact attributes; Apsis is read-mostly cache | Form submission processing and pre-filled form updates | Consent (subscription preferences) is bidirectional |
| Attribute changes flow CRM → Apsis only; never Apsis → CRM except consent | Integration architecture | Consent must sync bidirectionally for legal compliance |
| Main entities (Persons) downloaded from CRM; lead entities (Silhouettes) are not | CRM data synchronization lifecycle | Leads created only via form submission events |
| Enrichment attributes added via forms remain local to Apsis and do not sync to CRM | Pre-filled form submissions; integration architecture limitation | Integration does not support pushing profile updates to CRM |

### Form Syncing & Submissions

| Rule | Context | Exceptions |
|------|---------|-----------|
| Form syncing setup registers event listeners based on form type | Integration service form sync registration | Form tools: (open, submitted, started, viewed); form event tools: all events |
| Form submission validation requires at least one identification field (CRM ID, email, or SMS/phone) | Audience Subscription Worker processing | If none present, submission discarded silently as "useless" |
| Form submissions to CRM send only event notifications; do not send attribute data | Outbound Worker; integration architecture | CRM is master; CRM decides what to do with event notification |
| CRM response to form submission has three outcomes: matched_record, new_record, or empty | Outbound Worker response parsing; Merge Worker routing | Empty responses do NOT trigger merges; no profile consolidation occurs |

### Profile Management & Key Spaces

| Rule | Context | Exceptions |
|------|---------|-----------|
| When form submission results in CRM contact match, merge must occur between email/SMS key space and CRM key space | Merge Worker processing CRM response | If merge doesn't occur, duplicate profiles created in Apsis |
| Profile synced from CRM exists only in the CRM-specific key space unless explicitly merged | Delta Sync from CRM; pre-filled form profile resolution | If form submission creates profile in email key space without merging, duplicates occur |
| CRM ID field must never be modified on profile with active CRM integration | Form editing and pre-filled form submission | Modification breaks sync integrity and causes profile duplication |
| CRM ID is locked/hidden in form designer when integration is active | Form design time; prevents user misconfiguration | Must be implemented to prevent dangerous customer errors |
| Email field is locked (read-only) in pre-filled forms | Pre-filled form rendering | SMS-only profiles need verification of lock status |
| Each CRM installation gets unique key space to avoid entity collisions in multi-tenant environments | Key space generation; Keyspace function | Key space discriminator generated deterministically from section discriminator + CRM logical name |
| Keyspace discriminator is reproducible; same configuration always produces same discriminator | Key space management; CRM integration installation | Section discriminator must be stable; changes invalidate keyspace |

### Field Mapping & Data Protection

| Rule | Context | Exceptions |
|------|---------|-----------|
| Mapped fields should be protected from form overwrites when profile has CRM ID | Pre-filled form submissions; prevents sync discrepancies | Form tool queries Mappings Manager to determine protected fields |
| Mapped field overwrites by form submission will be reversed on next Delta Sync from CRM | Delta Sync worker overwrite behavior; CRM master principle | No conflict resolution; this is by design; users must understand |
| Enrichment attributes (unmapped fields) can be updated by forms and stored locally in Apsis | Form submission processing | These fields never sync to CRM; stay local only |
| Mapped field protection applies only to profiles with CRM ID attribute | Pre-filled form validation; identifies CRM-sourced profiles | Profiles without CRM ID are enrichable in all fields |

### Consent & Subscription Handling

| Rule | Context | Exceptions |
|------|---------|-----------|
| Consent changes in forms trigger bidirectional sync: form → Apsis → integration → CRM → Delta Sync back to Apsis | Pre-filled form consent submission | Only if subscription is in consent mapping; unmapped subscriptions are ignored |
| Consent is the ONLY attribute type that is bidirectionally synced between Apsis and CRM | Integration architecture; legal/compliance requirement | Exception to "CRM master" rule applies only to consent |
| Consent checkboxes in pre-filled forms should be pre-populated with profile's current consent status | Pre-filled form rendering | Currently not implemented (bug/feature gap); checkboxes are empty |

### Lead Entities & Entity Types

| Rule | Context | Exceptions |
|------|---------|-----------|
| Lead entities are created only when user submits Apsis form with minimum required data (email address) | Lead creation workflow; CRM connector behavior | Leads are not downloaded in bulk sync from CRM |
| Apsis must internally track and differentiate whether an entity is a Person or a Silhouette/Lead | Entity type management; profile update routing | Different behavior for each type (Persons updated; Leads not) |
| Profile updates are sent to CRM for main entities only; lead entities do not receive profile updates | CRM synchronization direction | Leads are signal-low, high-potential prospects; not enriched via sync |
| When form submission creates lead in CRM, CRM responds with silhouette/lead ID (not person ID) | Lead creation response handling; Merge Worker routing | Response format varies by CRM system; parser must handle both Person and Lead IDs |

---

## 6. Known Issues & Workarounds

| Issue | Severity | Component | Status | Workaround |
|-------|----------|-----------|--------|-----------|
| Pre-filled form consent checkboxes not pre-populated with profile's current consent status | Medium | Form Tool: Pre-filled Form Rendering | Open | Manually check/uncheck boxes or show current consent separately (not ideal) |
| Profile resolution for pre-filled forms submitted to CRM-key-space-only profiles may create duplicate profiles | High | Form Tool: Profile Resolution; Profile Identification System | Open | Manual merge of duplicates after submission (post-hoc recovery) |
| Form submissions can overwrite mapped CRM attributes, later reverted by Delta Sync, causing confusion | High | Form Tool: Form Submission Processing; Delta Sync Worker | Open | Implement field protection rule in form tool (query Mappings Manager) |
| CRM ID field may be selectable/editable in form designer when integration is active | High | Form Tool: Form Designer UI | Open | Hide CRM ID field entirely from form editor when integration active |
| Enrichment attributes added via forms are not synced back to CRM | Medium | Integration Pipeline: Architecture Decision | By Design | Accept that enrichment data stays local; do not expect in CRM |
| Consent changes only sync to CRM if subscription in consent mapping; unmapped subscriptions ignored | Medium | Integration Pipeline: Consent Mapping | By Design | Ensure all subscriptions used in forms are added to consent mapping |
| SMS-only profile lock status in pre-filled forms is unverified | Low | Form Tool: Pre-filled Form Field Locking | Unverified | Test SMS-only profiles to confirm field is locked |
| Hash algorithm for keyspace discriminator generation unconfirmed (SHA-1 or other?) | Medium | Keyspace function / Key Space Manager | Open | Document exact hash algorithm in connector implementation |
| Account section configuration structure and purpose not fully elaborated | Low | Account section configuration | Documentation Gap | Refer to integration configuration documentation or follow-up session |
| Profile update flow from CRM back to Apsis for person entities not detailed | Low | CRM connectors; Delta Sync | Documentation Gap | Refer to follow-up session or profile synchronization documentation |

---

## 7. Glossary

### Core Concepts

**Apsis Person** — A fully-qualified customer or prospect record in a CRM system with sufficient data. Called "Contact" in Microsoft Dynamics, "Person" in E-deal. These are main entities downloaded from CRM and receive profile updates via Delta Sync.

**Lead / Silhouette** — A lead entity representing a prospective customer with minimal data (typically form submission data). Called "Silhouette" in E-deal, "Lead" in Salesforce. Created when user submits Apsis form; does NOT receive profile updates after creation. Sales team notified for outreach.

**Keyspace** — A unique namespace identifier in Apsis that groups profiles by identifier type. Email keyspace uses email as key; SMS keyspace uses phone number; CRM keyspaces use CRM ID. Profiles can exist in multiple keyspaces and be merged.

**Key Space Merger / Profile Merge** — Operation consolidating two profiles from different keyspaces into single profile representation, ensuring lookups via either keyspace identifier return same profile ID.

**Keyspace Discriminator** — Unique identifier for a keyspace, generated deterministically from account section configuration (hashed) + CRM logical name. Format: `integration.keyspaces.[8-char-hash].[CRM_LOGICAL_NAME]`. Ensures reproducible, unique keyspaces per CRM installation.

**Section Discriminator** — Unique identifier within account configuration; hashed to produce part of keyspace discriminator. Must remain stable across connector installations.

**Mapped Fields** — Form fields with corresponding fields in CRM system (configured in Mappings Manager). CRM is source of truth; updates to mapped fields in CRM synced to Apsis via Delta Sync.

**Enrichment Attributes / Enrichment Fields** — Form fields WITHOUT corresponding CRM fields (unmapped). Data collected stays local to Apsis and never synced to CRM.

**Delta Sync** — Periodic synchronization pulling attribute changes from CRM to Apsis. Uses CRM ID as lookup key. Overwrites local Apsis values with CRM values (no conflict resolution).

**Consent Mapping** — Configuration mapping Apsis subscription/consent attributes to CRM subscription fields, enabling bidirectional sync of consent changes (only field type synced back to CRM).

**Identification Data** — Minimum required profile identification for form submission: CRM ID (if available), email address, or SMS/phone number. At least one must be present or submission discarded.

**CRM Master Principle** — Architectural rule that CRM is authoritative source of contact data. Apsis is read-mostly cache. Attribute updates flow CRM → Apsis only (except consent).

### Integration Components

**Form Syncing** — Feature enabling form submissions to trigger events sent to CRM systems. Form submission data converted to integration event format; CRM notified of activity (not attribute updates).

**Pre-filled Forms** — Forms sent to specific Apsis profiles (not public). Form fields automatically populated with profile's existing data. Upon submission, processed through standard form integration pipeline.

**Event Listeners** — Registry of events integration listens for when form synced to CRM. Form tools register (open, submitted, started, viewed); form event tools register all events in architecture.

**Audience Subscription Worker** — Message queue consumer that processes form submission events from Audience Service, extracts profile identification data, validates, converts to integration event format.

**Outbound Worker** — Integration component sending batched form events to CRM API. Receives CRM response (matched_record, new_record, or empty) and forwards to Merge Worker.

**Merge Worker** — Integration component processing CRM responses to form submissions. Performs profile merges between keyspaces when CRM returns contact ID.

**Mappings Manager Service** — Configuration service maintaining field mappings between Apsis and CRM for each integration. Queryable with account + section + integration ID to determine mapped fields.

**Profile Identification System** — Process locating correct profile in Apsis when form submitted or pre-filled form link opened. Must query appropriate keyspace (email, SMS, or CRM) to avoid duplicates.

**Base Installer** — Blank boilerplate class fulfilling all required functions for CRM connector implementations. Root of inheritance hierarchy; new functionality added once propagates to all connectors.

**Generic Connector** — Abstract connector implementation for standard CRM system types; inherits from Base Installer; specialized by CRM-specific connectors.

**CRM Connector** — Integration adapter for specific CRM system (Dynamics, E-deal, Salesforce, etc.); inherits from Base Installer or Generic Connector; handles CRM-specific entity mapping and API integration.

**Keyspace Function** — Function generating unique, deterministic keyspace identifiers per CRM installation. Defined on Base Installer; inherited by all connectors.

**Account Section** — Configuration section holding account-specific settings for CRM integration, including section discriminator used for keyspace generation.

**Integration Configuration** — Settings defining how Apsis connects to specific CRM system (CRM type, logical name, connection parameters).

### Data Flow Terms

**Form Submission Discrepancy** — Data inconsistency caused when form updates mapped attribute and CRM retains original value; mismatch until next Delta Sync (which overwrites form change with CRM value).

**Bootstrap / Bootstrapping** — Automatic creation of CRM-specific keyspace when CRM integration installed (e.g., "FSC Enterprise 12.1" keyspace).

---

## 8. Confidence Notes

### High Confidence ✓
- CRM master principle: CRM is authoritative source of contact attributes; Apsis is read-mostly cache
- Consent bidirectionality: consent/subscriptions are exception to unidirectional sync rule; require legal/compliance compliance
- Form submission event processing pipeline: Form Tool → Audience Service → Audience Subscription Worker → Kafka → Outbound Worker → CRM
- Key space architecture: multi-tenant isolation via unique keyspaces per CRM installation
- Lead entities: created via form submission, do not receive profile updates, different lifecycle than Persons
- Base Installer inheritance pattern: eliminates code duplication; new features propagate to all connectors
- CRM ID field protection: must never be editable to prevent sync mechanism breakage

### Medium Confidence ⚠️
- Mapped field protection implementation: form tool should query Mappings Manager, but exact call mechanism unclear
- Pre-filled form profile resolution: unresolved which keyspace queried to locate profile; duplicate prevention mechanism unclear
- Merge Worker error handling: retry logic and idempotency guarantees assumed but not detailed
- Delta Sync schedule: implied daily or hourly frequency, but exact value not documented
- Account section configuration structure: purpose clear (input to keyspace discriminator generation) but structure not detailed

### Low Confidence / Gaps ⚠️
- Hash algorithm for keyspace discriminator: assumed SHA-1 but unconfirmed; exact implementation not documented
- Form tool → Mappings Manager integration: query mechanism not specified; unclear how pre-filled form tool accesses mappings
- Profile update flow from CRM (Delta Sync) to Apsis: details of attribute update mechanism not documented
- Lead-to-person conversion workflow: CRM lifecycle for promoted leads not documented
- SMS-only profile field locking in pre-filled forms: email lock confirmed; SMS lock status unverified
- Form submission rate limiting: batch configuration (size, time window) not detailed
- Exact Kafka configuration: topic names, partition strategy, batch behavior not specified
- CRM API authentication per system: exact mechanism (API key, OAuth, etc.) not detailed; assumed per CRM type
- Consent field special handling: if mapped consent field is updated by form, ensure overwrites are allowed (not blocked by field protection)

### Conflicting / Unresolved Information
- **Profile resolution for pre-filled forms:** Session 1 discusses complexity of resolving profiles in CRM keyspace to prevent duplicates; mechanism unresolved. Which keyspace is queried? Does form tool auto-detect merge?
- **Merge Worker empty response handling:** Session 1 states empty CRM responses do NOT trigger merges; Session 2 implies profiles should be consolidated regardless. Clarification needed.
- **Lead-to-person conversion:** Session 2 introduces lead entity concept but doesn't explain workflow when sales qualifies lead and converts to customer. What happens to Silhouette/Lead ID?
- **Field source tracking audit:** Session 1 mentions "update existing profile data" validation setting; unclear if this refers to audit logs, metadata, or product-configurable toggle

### Documentation Gaps Requiring Follow-up
1. Exact hash algorithm for keyspace discriminator (SHA-1 confirmed?)
2. Account section configuration detailed structure and schema
3. Mappings Manager Service API specification (request/response format)
4. Pre-filled form profile resolution algorithm (which keyspace queried? merge detection?)
5. Delta Sync attribute update mechanism (partial update vs. full profile update)
6. Lead-to-person conversion workflow in CRM and Apsis
7. Consent field special handling (if mapped, should overwrites be blocked by field protection?)
8. SMS-only profile field locking verification
9. Form submission error responses (what errors returned to user?)
10. Bulk form submission behavior (rate limits, batch configuration, retry strategy)

---

## Unresolved Questions (Prioritized by Impact)

### Critical (Blocking Implementation)
1. **Pre-filled form profile resolution:** Which keyspace does form tool query to locate profile? How does it detect if profile exists in CRM keyspace? Does it auto-merge keyspaces if email matches CRM profile? This is critical for preventing duplicate profile creation.
2. **Mapped field protection implementation:** Exactly how and when does form tool call Mappings Manager to determine field protection rules? Before form submission or in worker? Does this happen for pre-filled forms?
3. **CRM ID field visibility in form designer:** Are CRM ID fields currently hidden/locked when integration is active, or is this a proposed future change? Needs implementation priority.

### High (Impacts Data Integrity)
4. **Merge Worker error recovery:** If profile merge fails (DB error, timeout), what happens? Is entire form submission retried? How many retries? Timeout values?
5. **Lead entity ID tracking:** When form creates lead in CRM and CRM responds with lead ID, how is this tracked in Apsis? Is it stored as different attribute than CRM Person ID? Can a Lead be converted to Person?
6. **Keyspace hash algorithm:** Exact algorithm (SHA-1, MD5, etc.) and implementation critical for reproducibility across installations.

### Medium (Affects UX / Reliability)
7. **Form submission rate limiting:** What is batching window? How many events per batch? What happens if batch fills before time window expires?
8. **Consent field special handling:** If mapped consent field, should field protection block overwrites? Or are consent fields excluded from protection?
9. **Delta Sync conflict resolution:** If CRM attribute changes while Apsis user modifies same attribute locally, and Delta Sync runs: what is the actual behavior? (Assume CRM wins, but confirm)
10. **SMS-only profile field locking:** Email is confirmed locked in pre-filled forms; is SMS/phone field also locked? Needs testing.

### Low (Documentation / Nice-to-Have)
11. Account section configuration detailed schema and examples
12. Lead-to-person conversion workflow (is this supported?)
13. Form submission audit logging (who made which changes?)
14. Salesforce Lead vs. Contact mapping details
15. Multi-CRM account support (can single Apsis account have multiple CRM integrations?)

---

## Key Decisions Requiring Confirmation / Clarification

| Decision | Current State | Required Confirmation |
|----------|---------------|----------------------|
| CRM ID field hidden in form designer when integration active | Proposed in Session 1 | Is this implemented? If not, when? High priority. |
| Mapped field protection query to Mappings Manager | Proposed in Session 1 | Who queries (form tool or worker)? When (form design time or submission)? How (API call, local cache)? |
| Pre-filled form consent checkbox pre-fill | Bug/gap identified in Session 1 | Is this a known bug? When will it be fixed? Critical for consent sync. |
| Hash algorithm for keyspace discriminator | SHA-1 assumed in Session 2 | Exact algorithm? Confirmed with code review? |
| Lead-to-person conversion workflow | Mentioned in Session 2 | Is this supported? What happens to Silhouette ID? When does conversion occur? |
| Multi-CRM multi-integration support | Implied by keyspace architecture | Can single account have multiple CRM integrations? Can form sync to multiple CRMs? |

---

## Architectural Patterns & Design Decisions

### Inheritance for Code Reuse
- **Base Installer** pattern: eliminates duplication across Dynamics, E-deal, Salesforce connectors
- New feature added to Base Installer automatically inherited by all connectors
- Reduces maintenance burden and ensures consistency

### Deterministic Keyspace Generation
- Keyspace discriminators reproducible across installations
- Same account section + CRM logical name → same discriminator
- Enables multi-tenancy without manual configuration

### Unidirectional Attribute Sync (with Consent Exception)
- CRM → Apsis only (Delta Sync)
- Apsis → CRM only for consent (bidirectional)
- Simplifies architecture; avoids sync complexity and data overwrites
- Trade-off: enrichment attributes don't appear in CRM (accepted by design)

### Event-Driven Form Submission Processing
- Form submission → Audience Service event → Audience Subscription Worker → Kafka batching → Outbound Worker → CRM
- Decoupled components; enables async processing and scaling
- Trade-off: eventual consistency; form submission not synchronous with CRM creation

### Lead Entity Distinction
- Leads created via form submission; don't receive profile updates
- Persons downloaded from CRM; receive updates via Delta Sync
- Allows Apsis to track signal quality and lifecycle differences

---

**END OF KNOWLEDGE BASE**