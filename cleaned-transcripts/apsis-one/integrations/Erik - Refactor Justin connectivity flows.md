---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One Integrations
topics: [Installation Flow Refactoring, Error Handling in Uninstallation, Function Responsibility Reduction, Webhook Management, Bootstrap Process, Connector Installation]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Installation Flow, Bootstrap Functions, Install Connector Function, Uninstall Connector Function, Webhook Management, API Credentials Storage, Generic Connector, Microsoft Dynamics, FSC Enterprise]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session covers a proposed refactoring of the **installation and uninstallation flows** in the Apsis One Integrations domain. Erik Andersson outlines the current complexity and error-handling problems, particularly around the `install_connector` and bootstrap functions, which have too many responsibilities nested within single functions. The core issue manifests during uninstallation when external API credentials have expired or been rotated, preventing cleanup of webhooks. The proposed solution is to flatten the nested function hierarchy by making the main installation flow the orchestrator, with explicit, separate calls for each responsibility (credential storage, webhook creation, bootstrap steps, etc.), allowing error handling to occur at the orchestration layer rather than deep in nested calls.

---

## Current Installation Flow Architecture

### Overview of Installation Steps

The installation process is described as **the most complex flow across all services** in the platform, comprising many sequential steps:

1. **Keyspace Creation**: For each CRM system installed, a corresponding keyspace is created (e.g., `dynamics_keyspace` for Microsoft Dynamics, `fsc_enterprise_keyspace` for FSC Enterprise)

2. **Bootstrap Schema Definition**: If a CRM system requires it, predefined attributes and events are created in Apsis. Erik notes:
   > "Some CRM systems have asked us like when you install this CRM then you should have these specific attributes and events created inside of Apsis and we do offer this functionality"

3. **Default Mappings Setup**: CRM identifiers are mapped to Apsis contact/person fields. Example: mapping CRM Contact ID to the configured ID field on contacts

4. **Resource Generation**: API keys and other resources are generated within Apsis and sent to the CRM system, along with:
   - Section discriminator
   - Section ID
   - Keyspace discriminator and ID
   
   These allow the CRM to utilize Apsis resources in custom flows

5. **Connectivity Registration**: Final database registration storing:
   - API URL
   - Credentials
   - (Configuration varies by CRM type)

---

## The Problem: Insufficient Granularity in Current Functions

### Root Issue: Monolithic Function Responsibilities

The main problem is that installation and uninstallation logic is consolidated into **single functions with too much responsibility**. If any step within a function fails, all subsequent steps are skipped.

[Erik Andersson]:
> "This flow today is not as granular as it should be. We have some steps of it where the functions are doing way too much. They have way too much responsibility and if one part in that function fails, like everything after it will not happen, and this is in particular prevalent for when you do uninstallations."

### The `install_connector` Function: A Case Study

For connectors like **FSC Enterprise** (generic connector), the `install_connector` function currently performs:

1. **Credential Storage**: Takes customer-entered API URL and API key from the UI and stores them in the database (e.g., `con_fsc` table for FSC connections)

2. **Webhook Creation**: Makes a request to the CRM system asking it to generate a webhook, receives the webhook ID, and stores it in the `webhooks` table with a foreign key link to the connectivity entry

#### The Uninstallation Problem

During uninstallation, the same flow is followed in reverse, but a critical issue emerges:

- **The API key may have expired or been rotated** between installation and uninstallation
- If the API key is invalid, the function **cannot make requests to the CRM system to delete webhooks**
- The current design **aborts the entire uninstallation process** if CRM communication fails
- Result: **Lingering webhooks remain in the CRM system** with no way to clean them up

[Erik Andersson]:
> "On uninstallations we would want to be able to remove everything inside of Apsis regardless sometimes if the CRM system returned an external error like the API keys that we have stored might already have expired or been rotated, so we cannot actually make any requests to the CRM system."

#### The Webhook Deletion Sequencing Challenge

The current logic must delete webhooks from the CRM *before* deleting them from the Apsis database. This is intentional:

- If webhooks are deleted from the Apsis database first but the CRM deletion fails, **there is no record of which webhooks to clean up on future attempts**
- However, if the CRM API is unavailable or credentials are invalid, **this forward-progress dependency blocks the entire uninstallation**

### Nested Function Problem with Error Flag Propagation

The installation flow uses an `ignore_external_errors` flag to allow proceeding despite CRM errors. However, because functions are deeply nested (function calls function calls function), this flag must be passed through multiple layers:

[Erik Andersson]:
> "Because these functions are so nested, like you have the install_connector which calls one function, which calls one function which calls one function, we would need to pass this flag on so deep and so many like it's so many nested requests that it becomes unmanageable after a while."

This makes the codebase fragile and difficult to maintain.

---

## Proposed Solution: Flattened Orchestration Model

### Core Principle: The Installation Flow as Orchestrator

The refactored approach moves the installation flow to the **top level**, where it explicitly orchestrates all sub-operations as separate, callable functions. Error handling happens at the orchestration layer, not in nested calls.

[Erik Andersson]:
> "What this flow aims to do is to lift, remove this nesting and make and make the installation installation flow be like the orchestrator of all of these things. So like in the installation flow you should make a explicit call to say I want to create the web hooks for this CRM system. You should not call like install_connector and inside install_connector you make the request. You do it in the actual installation flow because then you can handle all the errors in that main flow. You can choose to ignore those errors without having to pass any flags like 5 steps deeper in the flow."

### Bootstrap Function Decomposition

Instead of a single "God function" that creates folders, attributes, and events, split into:

- One explicit call to `bootstrap_attributes()`
- One explicit call to `bootstrap_events()`
- (Additional calls as needed)

Each call's errors are handled independently at the orchestration level.

### Connector Installation Decomposition

Instead of monolithic `install_connector()`, split into:

- One call to `store_credentials()` – stores API URL and key in the database
- One call to `create_webhooks()` – creates webhooks in the CRM and registers them locally

This allows, for example:
- Credential storage to succeed even if webhook creation fails
- Uninstallation to proceed even if the CRM webhook deletion API is unavailable
- Explicit error handling at each step without deep flag propagation

### Benefits

[Erik Andersson]:
> "I would not really agree, maybe as it is described, that this solution will prevent incidents that would be a bit taking it a bit too far maybe. However, it will by far simplify error handling for all of this because you can catch the errors and decide what to do with the error when you call like the create web hooks like do you want to proceed and delete things in the database or not?"

Key improvements:
1. **Simpler error handling**: Errors caught at orchestration level, not deep in call stacks
2. **Conditional cleanup**: Can decide per-step whether to continue on error
3. **Partial success**: Some steps can succeed while others fail gracefully
4. **No flag propagation**: Error handling logic stays local to the orchestration function
5. **Better uninstallation**: Can delete from Apsis database even if CRM communication fails (with proper sequencing)

---

## Scope: Generic Connector vs. Legacy Connectors

### Breadth of Impact

[Erik Andersson]:
> "The installation flow affects everyone because all of all of the connectors have like... all of them are split into at least two steps where like we create the web hooks in the second step and things can go wrong in the first one."

This refactoring applies to:
- **Generic Connector**
- **Microsoft Dynamics**
- **FSC Enterprise**
- Any other connector that stores API credentials and creates webhooks

### Full Sync Producer: Separate Concern

The **full sync producer** has different concerns:

- Not affected by credential storage issues since it's post-installation
- The difference between generic connector and legacy connectors (e.g., Dynamics) is which **endpoint** is called
- Legacy connectors (Dynamics) have one extra step: **mutate data to match Apsis formats**
  - Generic connector: "Give me data in the format we require"
  - Dynamics: "Get data from Dynamics endpoint" → "Transform to Apsis format"

[Erik Andersson]:
> "The full sync, however, is not as affected because the only real difference between the generic connector and the more legacy connectors like Dynamics is like which endpoint in the CRM system are we calling when when we retrieve the data."

Erik notes this is "rather stable and quite clear," so refactoring is less urgent here.

---

## Next Steps and Story Creation

### Epic and Story Planning

Erik intends to:
1. Create an **Epic** documenting the refactoring initiative
2. Create **Technical Stories** for each major breakdown, starting with at least two:
   - **Story 1**: Bootstrap function decomposition (split into three separate calls: folder, attributes, events)
   - **Story 2**: Connector installation decomposition (split credentials storage and webhook creation)

Additional stories will be identified upon deeper code review.

### Grooming Process

[Erik Andersson] and [Michal Rosikiewicz] agreed to:
- Erik creates the epic and initial problem statement
- Schedule a **grooming meeting** (estimate: 1–1.5 hours)
- Michal reviews stories beforehand to familiarize himself
- During the meeting, Erik will walk through the code to show exactly where these functions exist and how they interrelate
- Stories can be further refined with more sub-stories (e.g., for checking supported features, creating database entries) as needed

Tentative meeting: **Friday 11:00–12:00** (with option to extend if needed)

[Michal Rosikiewicz]:
> "I think you can create a meeting to groom them together, but at least I would like to have possibility to read them before the meeting and familiarize a bit with the details."

---

## Additional Considerations

### Pay-Duty Errors vs. Warnings

Erik mentions that some log levels can be converted from errors to warnings. This is a routine maintenance task and doesn't require extensive grooming. Different services handle this differently:
- **Integration service**: Triggers pay-duty alarms for Cloudwatch level 50 (error level)
- **MA service**: Similar approach
- Michal's service uses lock levels for errors and warnings in code; may configure level 50 thresholds in the future

This is deferred as lower priority compared to the installation flow refactoring.

### Stability Comparison

[Erik Andersson]:
> "The full sync producer is rather stable and quite clear. If if there were were to be any errors, you will easily see where those problems happen... whereas the full sync producer is rather stable and quite clear."

The installation flow refactoring is **more urgent** because it has known failure modes (expired credentials during uninstallation), whereas the full sync flow is stable.

---

## Key Takeaways

1. **Current State**: Installation flows in Apsis One Integrations are overly nested with multiple responsibilities per function, making error handling fragile and uninstallation problematic when external credentials expire.

2. **Core Problem**: When API credentials are rotated/expired, the CRM webhook cleanup fails, blocking the entire uninstallation process and leaving orphaned webhooks.

3. **Proposed Solution**: Flatten the function hierarchy by making the installation flow an explicit orchestrator that calls separate, single-responsibility functions (`store_credentials()`, `create_webhooks()`, `bootstrap_attributes()`, `bootstrap_events()`, etc.), with error handling at the orchestration level.

4. **Benefits**: Simplified error handling, conditional continuation on failure, no deep flag propagation, and ability to partially succeed (e.g., clean Apsis side even if CRM is unavailable).

5. **Scope**: The refactoring applies to all connectors (generic and legacy) but is most urgent for installation/uninstallation. Full sync producer refactoring is lower priority.

6. **Timeline**: Epic and stories to be created; grooming meeting scheduled for Friday 11:00–12:00 with code walkthrough to clarify exact function locations and interactions.

---

## Unresolved Questions

1. **Second story/diagram clarification**: Erik flagged that he doesn't fully understand what a second story (shown in a diagram) is intended to accomplish, as it may already be implemented. This requires further analysis before Friday's grooming meeting.

2. **Exact sub-story breakdown**: The full list of refactoring stories (e.g., checking supported features, creating database entries as separate stories) will be determined during code review before the grooming meeting.

3. **Log level threshold policy**: Whether Michal's service will align with Integration's pay-duty error threshold (level 50) for the foreseeable future remains an open decision.

---

## Action Items

- **Erik**: Review the installation flow code again; identify all refactoring opportunities; create Epic and initial story drafts; prepare code examples for grooming meeting
- **Erik**: Investigate the second diagram/story to clarify intent
- **Michal**: Review Epic and stories before Friday grooming meeting
- **Both**: Schedule grooming meeting for Friday 11:00–12:00 (with flexibility to extend)