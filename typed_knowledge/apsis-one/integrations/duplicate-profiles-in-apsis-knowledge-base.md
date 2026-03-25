---
title: Apsis One Integrations — duplicate profiles in apsis Knowledge Base
subdomain: duplicate profiles in apsis
generated: 2026-03-25T12:40:46.850Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (1 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — duplicate profiles in apsis Knowledge Base](#apsis-one-integrations-duplicate-profiles-in-apsis-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Form Tool](#form-tool)
    - [Audience Subscription Worker](#audience-subscription-worker)
    - [Event Format Converter](#event-format-converter)
    - [Kafka Queue (Batching)](#kafka-queue-batching)
    - [Outbound Worker](#outbound-worker)
    - [Merge Worker](#merge-worker)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Mappings Manager Service](#mappings-manager-service)
    - [Profile Identification System](#profile-identification-system)
    - [Key Space Manager](#key-space-manager)
    - [CRM Integration (Bootstrapped Key Spaces)](#crm-integration-bootstrapped-key-spaces)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling](#error-handling)
    - [Logging & Monitoring](#logging-monitoring)
    - [Deployment & Configuration](#deployment-configuration)
    - [Data Consistency & Transactions](#data-consistency-transactions)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [CRM Master Principle](#crm-master-principle)
    - [Profile Identification & Key Spaces](#profile-identification-key-spaces)
    - [Form Submission Validation](#form-submission-validation)
    - [Mapped Fields vs. Enrichment](#mapped-fields-vs-enrichment)
    - [Consent Handling](#consent-handling)
    - [Merge Operations](#merge-operations)
    - [Delta Sync & Attribute Updates](#delta-sync-attribute-updates)
    - [Pre-filled Forms](#pre-filled-forms)
    - [Event Listeners & Registration](#event-listeners-registration)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Issue 1: Pre-filled Form Consent Checkboxes Not Pre-populated](#issue-1-pre-filled-form-consent-checkboxes-not-pre-populated)
    - [Issue 2: Profile Resolution for Pre-filled Forms May Create Duplicates](#issue-2-profile-resolution-for-pre-filled-forms-may-create-duplicates)
    - [Issue 3: Form Overwrites of Mapped Fields Are Silently Reverted by Delta Sync](#issue-3-form-overwrites-of-mapped-fields-are-silently-reverted-by-delta-sync)
    - [Issue 4: CRM ID Field May Be Editable in Form Designer When Integration Active](#issue-4-crm-id-field-may-be-editable-in-form-designer-when-integration-active)
    - [Issue 5: Enrichment Attributes Are Not Synced to CRM](#issue-5-enrichment-attributes-are-not-synced-to-crm)
    - [Issue 6: Consent Changes Only Sync If Subscription Is in Consent Mapping](#issue-6-consent-changes-only-sync-if-subscription-is-in-consent-mapping)
    - [Issue 7: SMS-Only Profile Phone Field Lock Status in Pre-filled Forms Unverified](#issue-7-sms-only-profile-phone-field-lock-status-in-pre-filled-forms-unverified)
    - [Issue 8: Form Submission Validation Failure Is Silent](#issue-8-form-submission-validation-failure-is-silent)
    - [Issue 9: Profile Identification System May Not Cross-check Key Spaces](#issue-9-profile-identification-system-may-not-cross-check-key-spaces)
    - [Issue 10: Mappings Manager Query May Not Be Called by Form Tool](#issue-10-mappings-manager-query-may-not-be-called-by-form-tool)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence (Well-established, consistent across sources)](#high-confidence-well-established-consistent-across-sources)
    - [Medium Confidence (Documented but some ambiguities)](#medium-confidence-documented-but-some-ambiguities)
    - [Low Confidence (Unclear, contradictory, or needs verification)](#low-confidence-unclear-contradictory-or-needs-verification)
    - [Unresolved Disagreements](#unresolved-disagreements)
    - [Outdated/Needs Refresh](#outdatedneeds-refresh)
    - [Gaps in Knowledge](#gaps-in-knowledge)
  - [9. Confidence Summary Matrix](#9-confidence-summary-matrix)
  - [10. Recommended Pre-Release Testing Checklist](#10-recommended-pre-release-testing-checklist)
  - [End of Knowledge Base](#end-of-knowledge-base)

---

# Apsis One Integrations — duplicate profiles in apsis Knowledge Base

## 1. Subdomain Overview

The "duplicate profiles in apsis" subdomain encompasses the mechanisms, architecture, and business rules that prevent (or fail to prevent) duplicate profile creation when form submissions and CRM integrations interact. This subdomain covers form syncing to CRM systems, profile identification and resolution across multiple key spaces (email, SMS, CRM-specific), profile merging between key spaces, attribute synchronization from CRM (Delta Sync), and the protection of mapped CRM fields from form overwrites. The core tension is managing profiles that exist in different key spaces: when a form submission to a profile in the CRM-specific key space includes an email address, the system must correctly recognize that the email belongs to the existing profile and merge key spaces—not create a duplicate in the email key space.

## 2. Architecture Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    Form Submission Flow (Critical for Duplicates)│
└─────────────────────────────────────────────────────────────────┘

[Form Tool]
    ↓ (form submission event with identification data)
[Audience Service] → [Audience Subscription Worker]
    ↓ (extract CRM ID, email, phone; validate; convert format)
[Kafka Queue] (batching)
    ↓ (aggregate events)
[Outbound Worker]
    ↓ (send form event to CRM API; request contact match)
[CRM System] (FSC Enterprise 12.1, Dynamics, E-deal, etc.)
    ↓ (return response: matched_record / new_record / empty)
[Merge Worker]
    ↓ (if CRM returned contact ID: merge email/SMS key space with CRM key space)
[Key Space Manager]
    ↓ (consolidate profiles across key spaces)
[Profile Management]
    ↓ (result: single profile with multiple key space references OR duplicate profiles)

┌─────────────────────────────────────────────────────────────────┐
│             Delta Sync Flow (Attribute Updates from CRM)         │
└─────────────────────────────────────────────────────────────────┘

[CRM System] (source of truth)
    ↓ (contact attribute changes)
[Delta Sync Worker] (on schedule: daily/hourly)
    ↓ (pull delta changes via CRM API)
[Profile Identification System]
    ↓ (lookup profile using CRM ID in CRM-specific key space)
[Profile Management]
    ↓ (update attributes; OVERWRITE any local form modifications)
[Apsis Profile]

┌─────────────────────────────────────────────────────────────────┐
│                      Key Space Architecture                       │
└─────────────────────────────────────────────────────────────────┘

Email Key Space      SMS Key Space         CRM-Specific Key Spaces
  (email as key)      (phone as key)      (CRM ID as key; bootstrapped per integration)
                                          
Profile ID: abc123   Profile ID: def456   Profile ID: ghi789 (FSC Enterprise 12.1)
 └─ email            └─ phone             └─ CRM ID
                                          └─ email (if merged)
                                          └─ phone (if merged)

⚠️  MERGE OPERATION consolidates abc123 + ghi789 into single profile with both key space references

┌─────────────────────────────────────────────────────────────────┐
│            Integration Dependencies                              │
└─────────────────────────────────────────────────────────────────┘

Form Tool ←→ Mappings Manager Service  (query mapped fields for field protection)
Form Tool ←→ Profile Identification System  (resolve profile for pre-filled forms)
Merge Worker ←→ Key Space Manager  (initiate merges)
Delta Sync Worker ←→ Profile Identification System  (locate profiles by CRM ID)
Audience Subscription Worker ←→ Profile Identification System  (validate identification data)
```

## 3. Module Reference

### Form Tool

- **Purpose**: Create, manage, and configure forms; enable form syncing to CRM; handle form submissions and pre-filled form functionality.
- **Key files**: 
  - Form Designer
  - Form Event Listeners
  - Pre-filled Form Link Generation
  - Form Submission Processing Logic
- **Tech stack**: Form builder UI; integration service API; internal profile service API.
- **Data flow**:
  - **Inbound**: Form creation request (fields, configuration); form submission (field values, profile context); pre-filled form link access request.
  - **Outbound**: Form event to Audience Service (activity ID, event type, profile fields); query to Mappings Manager; query to Profile Identification System.
- **Business rules**:
  - CRM sync option must be selectable for forms.
  - Form submission must include at least one identification field (CRM ID, email, or SMS) or be discarded.
  - For profiles with active CRM integration: mapped fields should be protected from overwrites (proposal pending implementation).
  - CRM ID field must never be editable when integration is active.
  - Pre-filled form links must be locked to identified profile to prevent unauthorized access.
  - Consent checkboxes in pre-filled forms should display profile's current consent status (currently not implemented—known bug).
  - Email field is locked (read-only) in pre-filled forms; SMS-only profile lock status unverified.
- **Configuration**:
  - CRM Sync Option: enabled per form when target CRM selected.
  - Pre-filled Form Feature Flag: `'pre-filled-forms-enabled'` (currently restricted; being enabled for customers).
  - Form Field Lock Configuration: Email field locked; CRM ID hidden if integration active (proposed).
  - Event Listener Registration: Form tools register `[open, submitted, started, viewed]`; form event tools register all events in architecture.
- **Integration points**:
  - **Audience Service**: fire form submission event (activity ID, event type, profile data).
  - **Mappings Manager Service**: query mapped fields for field protection (account + section + integration ID).
  - **Profile Identification System**: resolve profile for pre-filled form validation.
  - **Integration Service**: register form sync listeners when CRM sync option selected.
  - **CRM Systems**: receive pre-filled form data via link; receive form event notification on submission.
- **Gotchas**:
  - ⚠️ **CRITICAL**: CRM ID field must NEVER be editable in form designer if integration exists. Modification breaks sync mechanism and causes duplicates. Currently may not be properly hidden.
  - ⚠️ If mapped field is overwritten by form submission, the overwrite is silently reverted on next Delta Sync with no warning to user.
  - Pre-filled form consent checkboxes are NOT currently pre-populated (bug).
  - SMS-only profile field lock status in pre-filled forms is unverified; needs testing.
  - Form submission validation is silent: if no identification data exists, submission is discarded with no error message to user.
  - Pre-filled form profile resolution must correctly identify profiles in CRM key space to avoid creating duplicates. This is unverified.
- **Tribal knowledge**:
  - The CRM ID is the "nuclear button" of the integration. If ever modified, the entire sync mechanism breaks. This MUST be prevented at all costs.
  - Pre-filled form feature adds significant complexity to profile resolution; critical failure mode if key space resolution is incorrect.
  - Form tool may not currently be querying Mappings Manager for each submission to determine field protection rules; this may need to be implemented.

---

### Audience Subscription Worker

- **Purpose**: Receive form submission events from Audience Service; extract profile identification data (CRM ID, email, phone); validate and convert to integration event format.
- **Key files**: 
  - Audience Subscription Worker Queue
  - Event Format Converter
- **Tech stack**: Message queue consumer (Kafka or similar); event transformation logic.
- **Data flow**:
  - **Inbound**: Form submission event from Audience Service (activity ID, event type, metadata, profile context).
  - **Outbound**: Converted integration event to Kafka Queue (CRM ID, email, phone, activity ID, timestamp, event type, profile key).
- **Business rules**:
  - Event requires at least one identification field: CRM ID OR email OR SMS/phone.
  - If no identification data, submission is silently discarded as "useless" (cannot attach to resource).
  - Extracts profile identification data and activity metadata but NOT attribute values from form submission.
  - Converts Audience event format to integration event format.
- **Configuration**: 
  - Identification Data Validation: requires at least one of CRM ID, email, SMS/phone.
- **Integration points**:
  - **Audience Service** (inbound): form submission event stream.
  - **Kafka Queue** (outbound): batching queue.
  - **Profile Identification System** (implicit): validates identification data format.
- **Gotchas**:
  - ⚠️ **SILENT FAILURE**: Public forms with only first/last name (no email/phone/CRM ID) are silently discarded. No error message to user or form tool.
  - Does NOT validate that identification data actually exists in Apsis; only validates format/presence.
  - Unclear whether it validates that email/phone uniquely identifies a profile or if merging is delegated to Merge Worker.
- **Tribal knowledge**:
  - This is the "gatekeeper" for form submissions into the integration pipeline. Many problematic submissions are rejected here silently.
  - The decision to require identification data is intentional: integration cannot send events to CRM for truly anonymous form submissions.

---

### Event Format Converter

- **Purpose**: Transform form submission events from Audience Service format to integration-specific format.
- **Key files**: Form submission event processing logic.
- **Tech stack**: Event transformation/mapping logic.
- **Data flow**:
  - **Inbound**: Raw Audience Service event.
  - **Outbound**: Integration format event (CRM ID, email, phone, activity ID, timestamp, event type, profile key).
- **Business rules**:
  - Extract only identification data and metadata; do NOT extract attribute field values.
  - Preserve activity ID and event type for CRM to use.
  - Include profile key or identifier to track which profile in Apsis triggered the event.
- **Configuration**: Event format mappings (likely hard-coded).
- **Integration points**:
  - **Audience Subscription Worker**: consumes raw event, produces converted event.
- **Gotchas**:
  - Format must be compatible with CRM API expectations; mismatch causes Outbound Worker failures.
  - Unclear how profile context is preserved if profile hasn't been merged across key spaces yet.
- **Tribal knowledge**:
  - This component enforces the "events only, no attributes" rule by omitting form field values from converted event.

---

### Kafka Queue (Batching)

- **Purpose**: Aggregate form submission events for batch processing to CRM; reduces individual API calls.
- **Key files**: Form event batching queue (topic configuration).
- **Tech stack**: Kafka message broker.
- **Data flow**:
  - **Inbound**: Individual integration-format events from Audience Subscription Worker.
  - **Outbound**: Batched events to Outbound Worker (batch size and timing TBD).
- **Business rules**:
  - Aggregate events into batches before forwarding to Outbound Worker.
  - Batch size and timing unclear; presumably optimized for CRM API throughput.
- **Configuration**: 
  - Batch size: not documented.
  - Batch timeout: not documented.
- **Integration points**:
  - **Audience Subscription Worker** (inbound): converted events.
  - **Outbound Worker** (outbound): batches.
  - **Kafka cluster**: storage and delivery guarantees.
- **Gotchas**:
  - ⚠️ Batching introduces latency between form submission and CRM notification. If customer expects real-time sync, this delay may be surprising.
  - Batch failure (e.g., Outbound Worker down) can accumulate events in Kafka; unclear if there's overflow protection.
  - Batch timeout unclear; events may wait in queue for extended period if volume is low.
- **Tribal knowledge**:
  - Batching is a performance optimization common in integration platforms. Exact batch parameters are not documented in this KT.

---

### Outbound Worker

- **Purpose**: Send batched form submission events to CRM system via HTTP API; receive and parse CRM response (matched contact, new contact, or empty).
- **Key files**: Outbound integration requests (HTTP client code).
- **Tech stack**: HTTP client library; REST API integration; response parsing.
- **Data flow**:
  - **Inbound**: Batch of integration-format events from Kafka Queue.
  - **Outbound**: HTTP POST to CRM API endpoint with batch of events.
  - **Return**: CRM response containing contact match/creation result with CRM ID (if applicable).
- **Business rules**:
  - Do NOT send attribute field values to CRM; only event notification (activity ID, event type, timestamps, identification data).
  - Send identification data (CRM ID, email, phone) to CRM for contact matching/creation.
  - CRM determines action (create new contact, match existing, or ignore).
  - Parse CRM response: `matched_record { crm_id }` or `new_record { crm_id }` or empty.
  - Forward CRM response to Merge Worker.
- **Configuration**: 
  - CRM API Request Format: HTTP POST with event payload (activity ID, event type, profile fields, timestamp).
  - CRM API Authentication: mechanism not detailed; varies by CRM system.
  - CRM API Endpoint URL: stored in integration configuration (not detailed).
- **Integration points**:
  - **Kafka Queue** (inbound): batched events.
  - **CRM Systems** (outbound): HTTP API POST.
  - **Merge Worker** (return): CRM response routing.
- **Gotchas**:
  - ⚠️ Request payload does NOT include attribute updates; CRM receives event notification only. If customer expects form data to be sent to CRM, they will be surprised.
  - CRM API errors (validation, authentication, rate limiting) must be handled and retried; unclear if this is implemented.
  - CRM response format must be exact; parsing errors could cause merge to be skipped silently.
  - Identification data in request must be formatted for CRM API (e.g., email lowercase, phone with country code); unclear if normalization occurs here.
- **Tribal knowledge**:
  - This is where the "CRM is master" principle is enforced: only events are sent, never attribute updates.
  - CRM response parsing is critical; if response is not parsed correctly, merge is skipped and duplicate profiles are created.

---

### Merge Worker

- **Purpose**: Process CRM response to form submission; determine if profile merge is needed; initiate key space consolidation when CRM returns contact ID.
- **Key files**: Merge worker queue; key space merge logic.
- **Tech stack**: Message queue consumer; profile merge orchestration; key space API.
- **Data flow**:
  - **Inbound**: CRM response from Outbound Worker (matched_record/new_record with CRM ID, or empty).
  - **Processing**: Look up profile in Apsis using profile key; add CRM ID attribute if not present; initiate merge between email/SMS key space and CRM-specific key space.
  - **Outbound**: Merge request to Key Space Manager.
- **Business rules**:
  - If CRM response includes contact ID (matched or new): add CRM ID attribute to profile and merge key spaces.
  - If CRM response is empty: no action; profile stays in original key space (email/SMS).
  - Merge must consolidate profiles so both key space lookups return same profile ID.
  - Merge must be idempotent (safe to retry).
  - Merge preserves original profile ID; adds CRM ID as attribute to same profile.
- **Configuration**: 
  - CRM Response Format Parsing: `matched_record { crm_id }`, `new_record { crm_id }`, empty.
- **Integration points**:
  - **Outbound Worker** (inbound): CRM response.
  - **Key Space Manager** (outbound): merge request.
  - **Profile Management** (inbound data source): profile lookup by profile key.
- **Gotchas**:
  - ⚠️ **CRITICAL**: If merge fails or doesn't occur, duplicate profiles are created—one in email key space, one in CRM key space. This is the PRIMARY CAUSE of duplicate profiles in the subdomain.
  - Merge must correctly identify which profile in Apsis corresponds to the CRM contact. If multiple profiles match (e.g., multiple profiles with same email), merge may consolidate wrong profiles.
  - Merge Worker must handle case where profile has already been merged (idempotency).
  - If CRM response parsing fails, entire response is lost and merge is never triggered (silent failure).
  - Merge operation may fail asynchronously (database error); no retry mechanism documented.
- **Tribal knowledge**:
  - Merge Worker is the "safety mechanism" that prevents duplicate profiles. If it fails or is skipped, duplicates are inevitable.
  - The decision to merge ONLY when CRM returns contact ID means profiles created in email key space without CRM contact match will never merge with CRM-synced profiles—they remain duplicates forever.

---

### Delta Sync Worker

- **Purpose**: Continuously synchronize contact attribute changes from CRM to Apsis; update profile data based on CRM changes; enforce CRM as source of truth.
- **Key files**: Delta sync scheduling; CRM API polling/webhook handler.
- **Tech stack**: Scheduled job (cron or similar); CRM API client; profile update logic.
- **Data flow**:
  - **Inbound**: Query CRM API for contact changes since last sync timestamp.
  - **Processing**: Locate profile in Apsis using CRM ID in CRM-specific key space; update attributes with CRM values.
  - **Outbound**: Profile attribute updates in Apsis.
- **Business rules**:
  - CRM is master; Apsis attribute values are always overwritten with CRM values (no conflict resolution).
  - Use CRM ID key space to locate profiles; do NOT search email/SMS key spaces.
  - Attribute updates are one-directional: CRM → Apsis only.
  - Do NOT update CRM ID attribute itself (immutable for sync purposes).
  - If attribute was modified locally in Apsis via form, CRM value OVERWRITES local value.
  - Sync is idempotent; safe to run multiple times.
- **Configuration**: 
  - Sync Schedule: frequency not documented; implied daily or hourly.
  - Attribute Overwrite Behavior: ALWAYS overwrite with CRM values (no override).
- **Integration points**:
  - **CRM Systems** (inbound): query delta changes via API.
  - **Profile Management** (outbound): attribute updates.
  - **Key Space Manager** (implicit): uses CRM ID to locate profiles in CRM-specific key space.
  - **Profile Identification System** (implicit): resolves CRM ID to profile ID.
- **Gotchas**:
  - ⚠️ **SILENT DATA LOSS**: If form submission modifies a mapped CRM attribute and Delta Sync runs immediately after, the form change is silently overwritten with CRM value. User sees their form submission rejected/reverted with no warning.
  - ⚠️ Profiles must have CRM ID in the CORRECT key space (e.g., FSC Enterprise 12.1). If CRM-synced contact exists only in email key space (no merge), Delta Sync will NOT find it using CRM ID and attribute sync will be missed.
  - Sync timestamp must be tracked correctly; drift could cause missed or duplicate updates.
  - CRM API query for delta must include all modified fields; incomplete delta results in data loss.
  - Enrichment attributes (unmapped fields) are never pulled from CRM (by design); these stay local to Apsis.
- **Tribal knowledge**:
  - Delta Sync is the "enforcer" of the CRM master principle. It ensures CRM always has the final say.
  - The fact that form submissions can be reverted by Delta Sync is intentional and by design, but creates poor UX if customers aren't aware.
  - Sync schedule is critical: too frequent = performance overhead; too infrequent = stale data in Apsis.

---

### Mappings Manager Service

- **Purpose**: Provide field mapping configuration between Apsis and CRM fields; enable form tool to determine which fields are mapped vs. enrichment fields; support dynamic field protection.
- **Key files**: Mappings Manager Service API; field mapping definitions (storage/database).
- **Tech stack**: Configuration service; REST API; configuration database.
- **Data flow**:
  - **Inbound**: Query with parameters (account ID, section ID, integration ID).
  - **Outbound**: List of mapped fields with CRM equivalents; field metadata (field type, read-only flag, etc.).
- **Business rules**:
  - Mapped fields: have CRM equivalents; synced from CRM via Delta Sync; should be protected from form overwrites.
  - Enrichment/unmapped fields: no CRM equivalent; stay local to Apsis; can be updated via forms.
  - Mappings are per integration (e.g., separate mappings for FSC Enterprise 12.1 vs. Dynamics).
  - Mappings include consent subscriptions (bidirectional sync).
- **Configuration**: 
  - Field Mapping Definitions: format unclear; likely includes Apsis field name, CRM field name, field type, read-only flag.
  - Consent Mapping: subscriptions that should sync bidirectionally to/from CRM.
- **Integration points**:
  - **Form Tool** (inbound query): to determine field protection rules.
  - **Integration Pipeline** (inbound query): to determine which fields to send to CRM.
  - **Delta Sync Worker** (inbound query): to determine which fields to pull from CRM.
- **Gotchas**:
  - ⚠️ Unclear how Form Tool accesses Mappings Manager. If form tool does NOT query for each submission, field protection will be ineffective or missing.
  - ⚠️ Consent Mapping is a separate configuration; unmapped subscriptions are ignored by integration (silent failure).
  - Field name matching must be exact (case-sensitive); mismatch will cause field to be treated as unmapped.
  - API response format is not documented; exact field names and structure are unknown.
  - Mappings must be updated manually if CRM field structure changes; no automatic detection of schema changes.
- **Tribal knowledge**:
  - Mappings Manager is the "source of truth" for field protection. Form tool MUST query this service to prevent overwrites.
  - Consent mappings are special case that enables bidirectional sync for compliance reasons.

---

### Profile Identification System

- **Purpose**: Locate profiles in correct key space (email, SMS, CRM ID); resolve profile identifiers for form submissions and pre-filled forms; prevent duplicate creation by correctly identifying existing profiles across key spaces.
- **Key files**: Profile resolution algorithm; key space routing logic.
- **Tech stack**: Profile lookup service; key space query API; merge detection logic.
- **Data flow**:
  - **Inbound**: Profile identifier (email, phone, CRM ID, profile key, or combination).
  - **Outbound**: Profile ID in correct key space; or "not found" + signal to create new profile.
- **Business rules**:
  - Given email, search email key space.
  - Given phone, search SMS key space.
  - Given CRM ID, search CRM-specific key space (e.g., FSC Enterprise 12.1).
  - If identifier found, return profile ID.
  - If identifier NOT found, return not-found signal (do not auto-create; let calling component decide).
  - For pre-filled forms: validate that profile exists and is accessible (security check).
  - For form submissions: allow creation of new profiles if no match found.
- **Configuration**: 
  - Key space routing: mappings of identifier type to key space name (implicit).
- **Integration points**:
  - **Form Tool** (inbound query): validate pre-filled form profile and retrieve current data.
  - **Audience Subscription Worker** (inbound query): validate identification data in form submission.
  - **Key Space Manager** (implicit outbound): query key spaces.
  - **Merge Worker** (implicit impact): affects whether profiles can be merged.
- **Gotchas**:
  - ⚠️ **CRITICAL FOR DUPLICATES**: If pre-filled form is submitted and profile exists only in CRM key space (via prior Delta Sync), Profile Identification System must correctly recognize the email belongs to that profile and trigger merge. If it doesn't, duplicate is created in email key space.
  - ⚠️ Unclear if system can detect that email in form submission matches email of profile in CRM key space. This is essential for preventing duplicates.
  - ⚠️ If profile exists in multiple key spaces due to incomplete merge, resolution may return wrong key space.
  - Email/phone lookups may return multiple profiles if duplicates already exist; conflict resolution strategy undefined.
  - For form submissions with CRM ID: must verify CRM ID is in correct key space; searching wrong key space will cause "not found" and lead to duplicate creation.
- **Tribal knowledge**:
  - Profile Identification System is the "first line of defense" against duplicates. If it fails to recognize that an identifier belongs to an existing profile, duplicates are created immediately.
  - The distinction between email key space lookup, SMS key space lookup, and CRM-specific key space lookup is non-obvious and easy to get wrong.
  - This system must be tested extensively with profiles in different key space combinations before pre-filled form release.

---

### Key Space Manager

- **Purpose**: Manage Apsis key spaces (email, SMS, CRM-specific key spaces); maintain key space registry; orchestrate profile merges between key spaces.
- **Key files**: Key space definitions; profile merge operations; key space bootstrap logic.
- **Tech stack**: Key space registry (database/config); merge orchestration logic; profile identity graph.
- **Data flow**:
  - **Inbound**: Merge requests (profiles to consolidate, source/target key spaces).
  - **Outbound**: Consolidated profiles with multiple key space references.
- **Business rules**:
  - Each key space is independent; profiles in one key space do NOT automatically appear in another.
  - CRM integration bootstrap creates CRM-specific key space (e.g., 'FSC Enterprise 12.1', 'dynamics').
  - CRM ID is immutable key in CRM-specific key space; must never be modified.
  - Profile merge consolidates two profiles from different key spaces into single identity.
  - After merge, both key space lookups return same profile ID.
  - Merge is initiated by Merge Worker; Key Space Manager executes.
  - Merge must preserve original profile ID; second profile is merged into first.
- **Configuration**: 
  - Key Space Definitions: email, SMS, per-CRM key spaces (bootstrapped on integration install).
  - CRM-Specific Key Spaces: FSC Enterprise 12.1, Dynamics, E-deal, etc. (created at integration bootstrap).
- **Integration points**:
  - **Merge Worker** (inbound): merge requests.
  - **CRM Integration Bootstrap** (implicit): creates CRM-specific key spaces.
  - **Delta Sync Worker** (implicit): uses CRM-specific key spaces to locate profiles.
  - **Profile Identification System** (outbound query): key space lookup.
- **Gotchas**:
  - ⚠️ If merge does NOT occur when it should (e.g., Merge Worker skipped), profiles remain in separate key spaces and appear as duplicates forever.
  - ⚠️ CRM key space creation must occur at bootstrap; if CRM integration is installed but key space not created, Delta Sync will fail to find profiles.
  - ⚠️ Profiles in email key space are NOT automatically checked against CRM key space during form submission. Duplicate creation requires explicit merge logic in Merge Worker, not automatic key space consolidation.
  - Merge operation is potentially expensive (database/graph operations); performance implications unknown.
  - Merge must be transactional (all-or-nothing); partial merges could create inconsistencies.
  - Merge must handle edge cases: profile already merged, merge of already-merged profiles, circular merge references.
- **Tribal knowledge**:
  - Key Space Manager is the "identity backbone" of the integration. Without proper key space management, duplicates are inevitable.
  - The fact that CRM-synced profiles live in separate key spaces (not email key space) is counter-intuitive and is the PRIMARY SOURCE OF DUPLICATES.
  - Merge operation is critical but complex; must be heavily tested and monitored.

---

### CRM Integration (Bootstrapped Key Spaces)

- **Purpose**: Create and maintain CRM-specific key spaces when CRM integration is installed; manage CRM ID as the identifying key for synced profiles.
- **Key files**: 
  - FSC Enterprise 12.1 key space (configuration/initialization)
  - Dynamics key space
  - E-deal key space
  - (and other CRM-specific key space definitions)
- **Tech stack**: Integration bootstrap logic; key space API; CRM configuration service.
- **Data flow**:
  - **Inbound**: CRM integration install request (account, section, CRM type).
  - **Outbound**: Bootstrapped key space (registered in Key Space Manager).
  - **Ongoing**: CRM contacts synced to this key space via Delta Sync.
- **Business rules**:
  - On CRM integration install: create key space named after CRM system (e.g., 'FSC Enterprise 12.1').
  - Key space uses CRM ID as the unique identifier.
  - All profiles synced from CRM exist in this key space.
  - Profiles synced from CRM do NOT automatically appear in email/SMS key spaces (unless merged).
  - Bootstrap is one-time operation; key space persists for lifetime of integration.
- **Configuration**: 
  - CRM Key Space Names: 'FSC Enterprise 12.1', 'dynamics', 'e-deal', etc. (per CRM system type).
  - CRM ID Field: name and format specific to each CRM system.
- **Integration points**:
  - **Key Space Manager** (outbound): register bootstrapped key space.
  - **Delta Sync Worker** (implicit): targets this key space for profile updates.
  - **Merge Worker** (implicit): merges email/SMS key space profiles into this key space.
- **Gotchas**:
  - ⚠️ If bootstrap fails or is skipped, CRM-specific key space may not exist, and Delta Sync will fail silently.
  - ⚠️ Key space name must be exact and consistent; mismatch between bootstrap name and Delta Sync query will cause failures.
  - ⚠️ CRM ID field name/format must be correct; mismatch with actual CRM ID format will cause lookup failures.
  - Key space creation is a one-time operation; re-bootstrap on integration update could be destructive.
  - If CRM integration is uninstalled and reinstalled, key space must be recreated or reused (unclear which is correct).
- **Tribal knowledge**:
  - CRM-specific key space creation is the foundation of the integration. Without it, all syncing fails.
  - The decision to create separate key spaces per CRM system (rather than shared key space with CRM-type-specific ID field) enables support for multiple CRM integrations but complicates profile resolution.

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **Form Tool**: Access control to form designer and pre-filled form links (locks link to profile ID to prevent unauthorized access).
- **CRM API Calls**: Integration must authenticate with CRM (mechanism varies by CRM system; not detailed in KT).
- **Mappings Manager Service**: Internal service-to-service authentication (not detailed).
- **Profile Access**: Pre-filled form links must be validated to ensure requester is authorized for that profile (email-based or SMS-based verification).

### Error Handling
- **Audience Subscription Worker**: Silent discard of submissions without identification data (no error message to user).
- **Form Tool**: Validation of mapped field overwrites (proposed); should reject or warn if mapped field is being overwritten.
- **Outbound Worker**: CRM API errors (validation, auth, rate limiting) must be handled; retry logic unclear.
- **Merge Worker**: Merge failures (database errors, profile resolution errors) should log but unclear if retry is triggered.
- **Delta Sync Worker**: Sync failures (API errors, profile not found) should log; unclear if sync is retried or skipped.
- **Mappings Manager Service**: Query failures should cause form submission to fail (fail-safe) or fall back to default behavior (unclear which).
- **Profile Identification System**: Lookup failures should signal "profile not found" (not auto-create); calling component decides action.

### Logging & Monitoring
- ⚠️ **CRITICAL**: Need audit logs for:
  - Form submissions (which fields changed, which field overwrites were blocked/allowed).
  - Merge operations (which profiles merged, success/failure).
  - Delta Sync (which attributes overwritten, conflicts detected, timestamp tracking).
  - Profile resolution (which key space was searched, duplicates detected).
- Monitoring should alert on:
  - Form submission discards (high volume suggests bad form design).
  - Merge failures (indicates duplicate profile risk).
  - Delta Sync failures (indicates stale data in Apsis).
  - CRM API errors (indicates integration health degradation).

### Deployment & Configuration
- **CRM Integration Installation**: Bootstrap creates key space; must be transactional and tested.
- **Form Sync Option Rollout**: Feature flag controls visibility; must be enabled per customer/section.
- **Pre-filled Form Feature Rollout**: Feature flag currently restricted; pending bug fixes before enabling.
- **Delta Sync Schedule**: Must be configurable per integration; defaults to daily or hourly (unclear).
- **Mappings Manager Configuration**: Must be updated for new CRM systems or field changes; currently manual process.

### Data Consistency & Transactions
- ⚠️ **CRITICAL**: Merge operations must be atomic. Partial merges could leave profiles in inconsistent state.
- Form submission event must be processed atomically: form event → identification validation → Kafka queue → Outbound Worker → CRM response → Merge Worker.
- Merge operation must be idempotent (safe to retry).
- Delta Sync must be idempotent; running multiple times should not cause duplicate updates.

---

## 5. Business Rules Reference

### CRM Master Principle
1. **CRM is authoritative source of contact data**. Apsis is read-mostly cache.
2. **Attribute data flows one-way: CRM → Apsis only**. Apsis never sends attribute updates to CRM (except consent).
3. **Form submissions notify CRM of events; do not send attribute data**. CRM decides what to do with the event.
4. **Consent is the only exception**: bidirectional sync between Apsis and CRM (for legal compliance).
5. **Delta Sync overwrites any local Apsis modifications with CRM values**. No conflict resolution.

### Profile Identification & Key Spaces
6. **Email key space uses email as unique identifier**. Profiles indexed by email address.
7. **SMS key space uses phone number as unique identifier**. Profiles indexed by phone number.
8. **CRM-specific key spaces use CRM ID as unique identifier**. Created automatically on CRM integration install (e.g., 'FSC Enterprise 12.1').
9. **Profiles can exist in multiple key spaces only via explicit merge**. Default: profile in one key space does NOT appear in another.
10. **Profile merge consolidates identity across key spaces**. After merge, both key space lookups return same profile ID.
11. **CRM ID must never be modified**. Modification breaks sync mechanism and causes profile duplication.
12. **Profiles synced from CRM exist only in CRM-specific key space by default**. They must be explicitly merged with email/SMS key spaces to consolidate identity.

### Form Submission Validation
13. **Form submission requires at least one identification field: CRM ID OR email OR phone**. Without any, submission is silently discarded.
14. **CRM ID field must never be editable/overwritable in form designer when integration is active**. Only through integration mechanism.
15. **Email field is locked (read-only) in pre-filled forms**. Users cannot change email to prevent unauthorized access.
16. **SMS-only profile phone field lock status is unverified**. Needs testing.

### Mapped Fields vs. Enrichment
17. **Mapped fields have CRM equivalents**. Configured in Mappings Manager; synced from CRM via Delta Sync.
18. **Enrichment attributes/unmapped fields have no CRM equivalent**. Stay local to Apsis; never synced to CRM.
19. **Mapped field overwrites by form submission are temporary**. Next Delta Sync from CRM reverts the overwrite (by design).
20. **Mapped field overwrites should be prevented by form tool when profile has CRM ID**. Requires query to Mappings Manager (proposed implementation).
21. **Form submissions can add enrichment attributes**. These stay in Apsis and are never pushed to CRM.

### Consent Handling
22. **Consent changes are bidirectional: Apsis ↔ CRM**. Unique exception to "CRM master" rule.
23. **Only subscriptions in consent mapping are synced**. Unmapped subscriptions are ignored by integration.
24. **Consent checkboxes in pre-filled forms should display profile's current consent status**. Currently not implemented (bug).
25. **Consent changes from forms trigger integration listeners**. If subscription is in consent mapping, change is sent to CRM.
26. **Delta Sync pulls updated consent from CRM back to Apsis**. Completes bidirectional cycle.

### Merge Operations
27. **Profile merge occurs when CRM form submission returns contact ID (matched or new)**. Merge Worker initiates.
28. **Merge consolidates email/SMS key space profile with CRM-specific key space profile**. Results in single identity.
29. **If CRM form submission returns empty response, no merge occurs**. Profile stays in original key space.
30. **Merge must be idempotent**. Multiple merge requests for same profiles must result in same outcome.
31. **If merge does not occur (Merge Worker failure), duplicate profiles result**. One in email key space, one in CRM key space. Very difficult to fix post-creation.

### Delta Sync & Attribute Updates
32. **Delta Sync runs on schedule (daily/hourly). Pulls attribute changes from CRM and updates Apsis profiles**.
33. **Delta Sync uses CRM ID in CRM-specific key space to locate profiles**. Email/SMS key space NOT used.
34. **Delta Sync overwrites Apsis attribute values with CRM values unconditionally**. No conflict resolution.
35. **If form submission modified mapped attribute and Delta Sync runs immediately after, form change is silently reverted**. By design.
36. **Profiles must have CRM ID in correct key space for Delta Sync to work**. If CRM-synced contact exists in email key space only (no merge), Delta Sync will not find it.
37. **Enrichment attributes are never pulled from CRM**. Stay local to Apsis.

### Pre-filled Forms
38. **Pre-filled form links must be locked to identified profile**. Cannot be reused by unauthorized users.
39. **Pre-filled form rendering must populate fields with profile's current data**. Consent checkboxes, attribute values, enrichment fields.
40. **Pre-filled form submission processed through standard form integration pipeline**. Same as public form submission but profile is pre-identified.
41. **Pre-filled form profile resolution must correctly identify profile in any key space**. Including CRM-specific key spaces (unverified).
42. **If pre-filled form is submitted to profile in CRM key space only, profile must be merged with email key space to consolidate**. If merge doesn't occur, duplicate is created.

### Event Listeners & Registration
43. **Form tools register event listeners for: open, submitted, started, viewed**. Subset of events.
44. **Form event tools register listeners for all events in architecture**. More comprehensive event set.
45. **Event listeners are registered at form sync setup time**. Based on form type selection.
46. **Event listeners are removed at form unsync time** (likely; not explicitly documented).

---

## 6. Known Issues & Workarounds

### Issue 1: Pre-filled Form Consent Checkboxes Not Pre-populated
- **Description**: Consent checkboxes in pre-filled forms display empty (unchecked) regardless of profile's current consent status.
- **Severity**: Medium (affects UX; users must manually verify/correct consent status).
- **Affected Component**: Form Tool: Pre-filled Form Rendering.
- **Root Cause**: Form rendering logic does not query profile's consent data before populating form.
- **Workaround**: Show current consent status separately (e.g., read-only field or note); ask user to manually check/uncheck consent boxes.
- **Fix**: Query profile's current consent status for each subscription and pre-populate checkboxes accordingly.
- **Test Case**: Submit pre-filled form to profile with known consent status; verify checkboxes match profile state.

---

### Issue 2: Profile Resolution for Pre-filled Forms May Create Duplicates
- **Description**: When pre-filled form is submitted to a profile that exists only in CRM key space (via prior Delta Sync), profile resolution may fail to recognize the email belongs to that profile, creating duplicate in email key space.
- **Severity**: **HIGH** (creates duplicate profiles; very difficult to fix post-creation).
- **Affected Components**: Form Tool: Profile Resolution; Profile Identification System; Merge Worker.
- **Root Cause**: Profile Identification System may only search email key space for email-based lookups; does NOT cross-check CRM key space.
- **Workaround**: Manual merge of duplicate profiles after creation (recovery, not preventive).
- **Fix**: Profile Identification System must:
  - Search email key space for email match.
  - If not found, search CRM key spaces for email attribute match (among synced profiles).
  - Return consolidated profile ID if found in either key space.
  - Alternatively, Merge Worker must detect duplicate and auto-merge.
- **Test Case**: (1) Create contact in CRM with email X. (2) Run Delta Sync to Apsis. (3) Generate pre-filled form for this profile. (4) Submit form with email X. (5) Verify single consolidated profile, not duplicate.
- **Status**: ⚠️ **UNVERIFIED** - must be tested before pre-filled form release.

---

### Issue 3: Form Overwrites of Mapped Fields Are Silently Reverted by Delta Sync
- **Description**: If pre-filled form changes a mapped field (e.g., first name) and Delta Sync runs immediately after, CRM value overwrites the form change with no warning to user.
- **Severity**: **HIGH** (data loss/reversion; poor UX; user may think form submission failed).
- **Affected Components**: Form Tool: Form Submission Processing; Delta Sync Worker.
- **Root Cause**: Architectural decision: CRM is master; Delta Sync always overwrites local changes. No conflict resolution.
- **Workaround**: Prevent overwrites by form tool (proposed feature; not yet implemented). Query Mappings Manager for mapped fields; block updates if profile has CRM ID.
- **Fix**: Form Tool should:
  - Query Mappings Manager for list of mapped fields.
  - For profiles with CRM ID: reject updates to mapped fields; allow updates to enrichment fields only.
  - Return validation error to user explaining why field update is blocked.
- **Test Case**: (1) Submit pre-filled form updating first_name (mapped field) on profile with CRM ID. (2) Verify form submission rejected or warning shown. (3) Run Delta Sync. (4) Verify first_name is unchanged (CRM value).
- **Status**: ⚠️ **PROPOSED** - not yet implemented.

---

### Issue 4: CRM ID Field May Be Editable in Form Designer When Integration Active
- **Description**: Form designer may allow selecting CRM ID as a form field; if form is submitted with updated CRM ID, sync mechanism breaks.
- **Severity**: **HIGH** (breaks entire integration; causes profile duplication and loss of sync integrity).
- **Affected Component**: Form Tool: Form Designer UI.
- **Root Cause**: Form designer does NOT prevent CRM ID selection when integration is active.
- **Workaround**: Manually ensure CRM ID is not included in form fields (requires user discipline; not reliable).
- **Fix**: Form Designer must hide CRM ID field from selection if CRM integration is active on the section. Alternatively, prevent any updates to CRM ID field in form submission processing.
- **Test Case**: (1) Create form with CRM integration active. (2) Attempt to add CRM ID as a field. (3) Verify field is hidden/unavailable. (4) If already in form, verify submission rejects CRM ID updates.
- **Status**: ⚠️ **DECISION**: CRM ID field must NEVER be editable in form when integration exists (high confidence).

---

### Issue 5: Enrichment Attributes Are Not Synced to CRM
- **Description**: Form fields that are not mapped to CRM (enrichment attributes) are collected in Apsis but never synced back to CRM. Customers may expect these to appear in CRM.
- **Severity**: Medium (architectural limitation; not a bug per se, but limitation of current integration design).
- **Affected Component**: Integration Pipeline (architecture design).
- **Root Cause**: Integration architecture does not support pushing profile attribute updates from Apsis to CRM.
- **Workaround**: Accept enrichment data stays local to Apsis; do not expect it in CRM. Manually sync enrichment data to CRM via separate process if needed.
- **Fix**: Implement reverse sync from Apsis to CRM (significant effort, not currently prioritized).
- **Test Case**: (1) Submit pre-filled form updating unmapped enrichment field. (2) Verify update is stored in Apsis. (3) Query CRM for this contact. (4) Verify enrichment field is NOT present in CRM (expected behavior).
- **Status**: ⚠️ **BY DESIGN** - documented limitation of integration architecture.

---

### Issue 6: Consent Changes Only Sync If Subscription Is in Consent Mapping
- **Description**: If form submission changes consent for a subscription that is NOT in the consent mapping configuration, the integration does NOT register a listener, and the consent change is ignored (never synced to CRM).
- **Severity**: Medium (silent failure; customer may not realize consent wasn't synced to CRM).
- **Affected Component**: Integration Pipeline: Consent Mapping; Merge Worker.
- **Root Cause**: Event listeners are only registered for subscriptions in consent mapping. Unmapped subscriptions are silently ignored.
- **Workaround**: Ensure all subscriptions used in forms are added to consent mapping configuration.
- **Fix**: Form Tool should warn if form includes consent field for unmapped subscription. Integration should allow listening for unmapped subscriptions (if compliance allows).
- **Test Case**: (1) Create form with consent checkbox for unmapped subscription. (2) Submit form changing consent for unmapped subscription. (3) Query CRM for contact's subscription status. (4) Verify subscription status in CRM is unchanged (expected behavior, but may surprise customer).
- **Status**: ⚠️ **KNOWN** - documented constraint of integration design.

---

### Issue 7: SMS-Only Profile Phone Field Lock Status in Pre-filled Forms Unverified
- **Description**: Pre-filled forms lock the email field to prevent unauthorized access via email modification. For SMS-only profiles (identified by phone number, no email), it is unclear whether the phone field is also locked.
- **Severity**: Low (security concern if phone field is not locked; needs verification).
- **Affected Component**: Form Tool: Pre-filled Form Field Locking.
- **Root Cause**: Field lock implementation may be hard-coded for email field only.
- **Workaround**: None; needs testing to determine current behavior.
- **Fix**: Form Tool must lock SMS/phone field for SMS-only profiles same as email field is locked.
- **Test Case**: (1) Create SMS-only profile (phone = 1234567890, no email). (2) Generate pre-filled form link for this profile. (3) Open link and verify phone field is read-only/locked. (4) Attempt to modify phone field; verify changes are rejected or not saved.
- **Status**: ⚠️ **UNVERIFIED** - must be tested.

---

### Issue 8: Form Submission Validation Failure Is Silent
- **Description**: If public form submission contains no identification data (CRM ID, email, or SMS), the entire submission is silently discarded by Audience Subscription Worker. User receives no error message.
- **Severity**: Medium (poor UX; user doesn't know submission failed).
- **Affected Component**: Form Tool; Audience Subscription Worker.
- **Root Cause**: Validation discards submission without notifying form tool or user.
- **Workaround**: Ensure form requires at least one identification field; design form to make email mandatory for public forms.
- **Fix**: Form submission validation should return error to user explaining required fields. Form Tool should warn if form could be submitted without identification data.
- **Test Case**: (1) Create public form with only first/last name fields (no email, no phone). (2) Submit form with name only, no email. (3) Verify error message shown to user (or form tool prevents submission with warning).
- **Status**: ⚠️ **BY DESIGN** - intentional (can't sync anonymous forms to CRM), but UX is poor.

---

### Issue 9: Profile Identification System May Not Cross-check Key Spaces
- **Description**: When form is submitted with email, Profile Identification System may only search email key space; if profile exists in CRM key space (synced from CRM, not yet merged), system will not find it, leading to duplicate creation.
- **Severity**: **HIGH** (duplicate profiles).
- **Affected Component**: Profile Identification System; Merge Worker.
- **Root Cause**: Key space resolution logic may not check all key spaces for identifier match.
- **Workaround**: Ensure all profiles created in CRM are first synced to Apsis via Delta Sync (creating CRM key space entry), then trigger form submissions (which will merge). Avoid form submissions before Delta Sync.
- **Fix**: Profile Identification System must:
  - Query email key space for email match.
  - If not found, query all CRM-specific key spaces for email attribute.
  - Return consolidated profile ID from any key space.
  - Merge key spaces if profile found in multiple spaces.
- **Test Case**: (1) Create contact in CRM with email X. (2) Sync to Apsis via Delta Sync (exists in CRM key space). (3) Submit form with email X. (4) Verify single profile, not duplicate.
- **Status**: ⚠️ **CRITICAL UNVERIFIED** - must be fixed before pre-filled form release.

---

### Issue 10: Mappings Manager Query May Not Be Called by Form Tool
- **Description**: Form Tool may not query Mappings Manager Service to determine field protection rules. If hard-coded field names are used instead, dynamic field protection is not possible.
- **Severity**: Medium (field protection ineffective if mappings change or vary by customer).
- **Affected Component**: Form Tool; Mappings Manager Service.
- **Root Cause**: Form Tool integration with Mappings Manager may not be implemented.
- **Workaround**: Hard-code field names for protection (fragile; breaks if CRM mappings change).
- **Fix**: Form Tool must query Mappings Manager for each account + section + integration ID to retrieve mapped fields dynamically.
- **Test Case**: (1) Configure different field mappings for two integrations. (2) Create form for each integration. (3) Verify form submission blocking behavior matches each integration's mappings (not identical).
- **Status**: ⚠️ **UNCLEAR** - must verify form tool is calling Mappings Manager.

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Apsis One** | Customer data platform (CDP) and marketing automation system. Maintains profiles, audiences, subscriptions. |
| **CRM** | Customer Relationship Management system. External system (FSC Enterprise, Dynamics, E-deal, etc.) that is authoritative source of contact data. |
| **Key Space** | Namespace in Apsis that groups profiles by unique identifier type (email, SMS phone, CRM ID). Profiles can exist in multiple key spaces and be merged. |
| **Email Key Space** | Key space indexed by email address. Standard key space where email-identified profiles live. |
| **SMS Key Space** | Key space indexed by SMS/phone number. Where phone-identified profiles live. |
| **CRM-Specific Key Space** | Key space created for each CRM integration (e.g., 'FSC Enterprise 12.1'). Uses CRM ID as unique key. Profiles synced from CRM exist here. |
| **Profile Merge** | Operation consolidating two profiles from different key spaces into single identity. After merge, both key space lookups return same profile ID. |
| **Bootstrapped Key Space** | CRM-specific key space automatically created when CRM integration installed. Example: 'FSC Enterprise 12.1'. |
| **Mapped Fields** | Form/profile fields that have corresponding fields in CRM system. Configured in Mappings Manager. Synced from CRM via Delta Sync. |
| **Enrichment Attributes** | Form/profile fields with no CRM equivalent (unmapped). Collected in Apsis; never synced to CRM. |
| **Delta Sync** | Periodic synchronization of attribute changes from CRM to Apsis. Runs on schedule (daily/hourly). Pulls CRM changes and overwrites Apsis attributes. |
| **Form Syncing** | Feature enabling form submissions to trigger events sent to CRM systems. Form submission data converted to integration event format; CRM notified of activity. |
| **Pre-filled Forms** | Forms sent to specific Apsis profiles (not public). Form fields automatically populated with profile's existing data. Submission processed through standard integration pipeline. |
| **Consent Mapping** | Configuration mapping Apsis subscription/consent attributes to CRM subscription fields. Enables bidirectional sync of consent changes (only field type synced back to CRM). |
| **Identification Data** | Minimum required profile identification for form submission: CRM ID, email, OR phone. At least one must be present. |
| **Form Event Listeners** | Registry of events integration listens for when form synced to CRM. Form tools: [open, submitted, started, viewed]. Form event tools: all events. |
| **Audience Subscription Worker** | Message queue consumer processing form submission events from Audience Service. Extracts identification data; converts to integration format. |
| **Outbound Worker** | Integration component sending batched form events to CRM API. Receives CRM response (matched/new contact or empty). |
| **Merge Worker** | Integration component processing CRM responses. Performs profile merges when CRM returns contact ID. |
| **Mappings Manager Service** | Configuration service maintaining field mappings between Apsis and CRM per integration. Queryable to determine mapped vs. enrichment fields. |
| **Profile Identification System** | Service locating profiles in correct key space. Resolves profile identifiers for form submissions and pre-filled forms. Prevents duplicate creation. |
| **Key Space Manager** | Service managing Apsis key spaces; orchestrates profile merges between key spaces. |
| **CRM Master Principle** | Architectural rule: CRM is authoritative source of contact data. Apsis is read-mostly cache. Attribute updates flow CRM → Apsis only (except consent). |
| **Form Submission Discrepancy** | Data inconsistency when form updates mapped attribute and CRM retains original; mismatch until Delta Sync overwrites form change with CRM value. |
| **Profile Resolution** | Process of locating correct profile in Apsis. Must query appropriate key space (email, SMS, or CRM) to avoid duplicates. |
| **CRM ID** | Unique identifier in CRM system. Immutable within Apsis integration (modification breaks sync; causes duplicates). |
| **Duplicate Profiles** | Two or more profile records in Apsis representing same person. Occurs when merge doesn't happen or profile resolution fails. |
| **Matched Record** | CRM response indicating existing contact matched by email/phone/other identifier. Returns contact ID for merge. |
| **New Record** | CRM response indicating new contact created for form submission. Returns contact ID for merge. |
| **Empty Response** | CRM response indicating event received but no action taken (no contact match/create). No merge triggered. |
| **Consent** | Subscription preference (opt-in/opt-out). Only attribute type bidirectionally synced between Apsis and CRM. |
| **Field Protection** | Mechanism preventing form submissions from overwriting mapped CRM fields when profile has CRM ID. Prevents data loss from Delta Sync reversion. |
| **Sync Integrity** | State where profiles across key spaces are correctly merged and CRM ID is valid and immutable. Broken if CRM ID is modified. |

---

## 8. Confidence Notes

### High Confidence (Well-established, consistent across sources)
- ✅ **CRM Master Principle**: CRM is authoritative source. Apsis is cache. Architecture is well-defined.
- ✅ **Key Space Concept**: Email, SMS, CRM-specific key spaces are real. Separation is intentional.
- ✅ **Consent Bidirectionality**: Consent is only attribute synced back to CRM (by design, for compliance).
- ✅ **Form Submission Flow**: Audience Service → Kafka → Outbound Worker → CRM → Merge Worker → Key Space Manager.
- ✅ **Delta Sync Behavior**: Overwrites local Apsis attributes with CRM values unconditionally. Runs on schedule.
- ✅ **CRM ID Immutability**: CRM ID must never be modified. Modification breaks sync.
- ✅ **Mapped vs. Enrichment Fields**: Distinction is clear. Mapped fields sync from CRM. Enrichment fields stay local.

### Medium Confidence (Documented but some ambiguities)
- ⚠️ **Profile Identification System**: Understanding of how it resolves profiles across key spaces is indirect. Document claims it prevents duplicates, but mechanism is unclear.
- ⚠️ **Form Tool Integration with Mappings Manager**: Proposed that form tool queries Mappings Manager for field protection, but unclear if this is currently implemented.
- ⚠️ **Merge Idempotency**: Stated that merge must be idempotent, but implementation details not provided.
- ⚠️ **Pre-filled Form Profile Resolution**: Critical feature for preventing duplicates, but testing is incomplete and mechanism is unclear.
- ⚠️ **SMS Field Lock Status**: Email field confirmed locked in pre-filled forms, but SMS field lock status unverified.
- ⚠️ **Consent Checkbox Pre-fill**: Stated as bug/gap; work-around described, but fix timeline unknown.
- ⚠️ **Batch Timing**: Kafka batching parameters (size, timeout) not documented. Performance implications unknown.

### Low Confidence (Unclear, contradictory, or needs verification)
- ❌ **Form Tool Callpath for Mappings Manager**: Unclear whether form tool currently calls Mappings Manager or if this is proposed future behavior.
- ❌ **Profile Resolution Cross-Key-Space Search**: Claims Profile Identification System prevents duplicates, but doesn't clearly explain if it searches CRM key spaces when email is provided.
- ❌ **Merge Worker Recovery**: What happens if merge fails (database error, network timeout)? Is retry triggered? Timeout?
- ❌ **CRM API Response Format Parsing**: Exact format of matched_record and new_record responses not documented. Hard to implement without seeing actual examples.
- ❌ **Mappings Manager API Response Format**: Exact fields and structure of mapping response not documented.
- ❌ **Event Listener Removal**: Stated that listeners are registered at sync setup, but unclear if removed at unsync time.
- ❌ **Multi-CRM Scenarios**: Unclear how system behaves if multiple CRM integrations are active. Separate key spaces per CRM? How are profiles merged across multiple CRMs?
- ❌ **Profile Already Merged Scenarios**: If profile is already merged and merge is requested again, what happens? Idempotency mechanism unclear.
- ❌ **Form Tool Rejection of Mapped Field Overwrites**: Proposed feature; implementation status, UX, and error messages unclear.

### Unresolved Disagreements
None documented in single KT session. However, several unresolved questions remain:
- **Key Space Querying Strategy**: Should Profile Identification System query all key spaces for a given identifier, or only the expected key space? Current behavior unclear.
- **Merge Trigger Condition**: Should merge occur only when CRM returns contact ID (current), or should system auto-detect duplicates and merge (alternative)?
- **Field Protection Scope**: Should only mapped fields be protected, or also CRM ID? (Decision: CRM ID must never be editable; mapped field protection is proposed.)
- **Enrichment Data Lifecycle**: If enrichment attribute is added via form, then later a CRM field mapping is created for that attribute, what happens on next Delta Sync? Expected behavior unclear.

### Outdated/Needs Refresh
- **Consent Checkbox Pre-fill Status**: Documented as not implemented (bug). Unclear if fixed since KT session.
- **Pre-filled Form Feature Status**: Documented as "hidden/restricted" pending bug fixes. Unclear what fixes have been completed.
- **Profile Merge Test Coverage**: No recent test results provided. Critical scenario (pre-filled form to CRM-synced profile) is marked "unverified".

### Gaps in Knowledge
1. **CRM-Specific Implementation Details**: How does form event format differ between FSC Enterprise 12.1, Dynamics, and E-deal? Not documented.
2. **Performance Characteristics**: How many profiles can be merged per second? What is Kafka batch throughput? What is Delta Sync performance at scale?
3. **Rollback & Recovery Procedures**: If duplicate profiles are created, how are they recovered? Manual merge? Automated cleanup? Unclear.
4. **Multi-Account Multi-Tenant**: How does key space isolation work across customer accounts? Are CRM-specific key spaces per-account? Unclear.
5. **Regulatory Compliance**: Are audit logs required for GDPR/CCPA? Is field-level change tracking required? Unclear.
6. **Testing Strategy**: What test coverage exists for duplicate prevention? Critical paths tested? Unverified.

---

## 9. Confidence Summary Matrix

| Topic | Confidence | Status | Priority |
|-------|-----------|--------|----------|
| **CRM Master Principle** | ✅ High | Stable | Core |
| **Key Space Architecture** | ✅ High | Stable | Core |
| **Profile Merge Mechanism** | ⚠️ Medium | Proposed features pending | Critical |
| **Pre-filled Form Duplicate Prevention** | ❌ Low | Unverified | **BLOCKING** |
| **Profile Identification Cross-Key-Space** | ❌ Low | Unclear | **BLOCKING** |
| **Form Tool Mapped Field Protection** | ⚠️ Medium | Proposed (not implemented) | High |
| **Mappings Manager Integration** | ⚠️ Medium | Likely implemented, UX unclear | High |
| **Delta Sync Behavior** | ✅ High | Stable | Core |
| **Consent Bidirectional Sync** | ✅ High | Stable (checkbox bug pending fix) | High |
| **Form Submission Validation** | ✅ High | Stable (silent failure by design) | Medium |
| **CRM API Response Handling** | ⚠️ Medium | Likely implemented, format not documented | Medium |
| **Merge Idempotency** | ⚠️ Medium | Stated requirement, implementation unclear | High |
| **SMS Field Lock in Pre-filled Forms** | ❌ Low | Unverified | Low |

---

## 10. Recommended Pre-Release Testing Checklist

Before pre-filled forms feature release, must test:

- [ ] **Test 1: Pre-filled form submitted to profile in CRM key space only**
  - Create contact in CRM with email X, name Y.
  - Run Delta Sync. Verify contact exists in CRM key space, NOT email key space.
  - Generate pre-filled form for this profile.
  - Submit form with no changes. Verify single profile ID (merged across key spaces), not duplicate.
  - **MUST PASS** before release.

- [ ] **Test 2: Pre-filled form modifying enrichment field on CRM-synced profile**
  - Create contact in CRM with name Y.
  - Sync to Apsis.
  - Submit pre-filled form adding enrichment field (unmapped).
  - Verify enrichment field is stored in Apsis.
  - Verify enrichment field does NOT appear in CRM.
  - **Expected behavior verification**.

- [ ] **Test 3: Pre-filled form modifying mapped field on CRM-synced profile**
  - Create contact in CRM with first_name = "John" (mapped field).
  - Sync to Apsis.
  - Submit pre-filled form changing first_name to "Jane".
  - Verify field protection blocks update (or warns user).
  - Run Delta Sync. Verify first_name reverts to "John".
  - **MUST PASS** before release (if field protection implemented).

- [ ] **Test 4: Consent checkbox pre-fill**
  - Create profile with known consent status (opted-in to Newsletter).
  - Generate pre-filled form with consent checkbox.
  - Verify checkbox is pre-filled (checked).
  - Uncheck and submit.
  - Verify consent updated in Apsis and synced to CRM.
  - **MUST PASS** before release (requires checkbox bug fix).

- [ ] **Test 5: SMS-only profile field lock**
  - Create SMS-only profile (phone = 1234567890, no email).
  - Generate pre-filled form link.
  - Open link and verify phone field is read-only/locked.
  - Attempt to modify phone. Verify changes rejected or not saved.
  - **MUST VERIFY** before release.

- [ ] **Test 6: CRM ID field hidden in form designer**
  - Create form with CRM integration active.
  - Attempt to add CRM ID field in form designer.
  - Verify field is NOT available/hidden.
  - **MUST PASS** before release.

- [ ] **Test 7: Duplicate detection in production-like data**
  - Create 100 contacts in CRM with emails in various formats (uppercase, lowercase, with/without spaces).
  - Sync all to Apsis.
  - Submit pre-filled forms for 50% of profiles.
  - Verify no duplicates created (email+CRM key space correctly merged).
  - Run Delta Sync. Verify all profiles still correctly merged.
  - **Load and stress test**.

---

## End of Knowledge Base

**Last Updated**: Aggregated from Erik - CRM and pre-fill form.md KT session.
**Confidence Assessment**: See Section 8 for confidence levels and unresolved items.
**Critical Blockers**: Pre-filled form duplicate prevention (Tests 1, 4, 5, 6 above) must be verified before release.