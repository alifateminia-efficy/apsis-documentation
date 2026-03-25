---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One Integrations
topics: [Installation Flow Refactoring, Error Handling, Nested Function Architecture, Webhook Management, Uninstallation Flow, Bootstrap Process, Connector Installation]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Installation Flow, Uninstall Connector, Install Connector, Bootstrap Functions, Webhook Management, API Credentials, Generic Connector, Microsoft Dynamics, FSC Enterprise]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Different Types of Connectors, Inbound Flow, Outbound Flow, Microsoft Dynamics]
---

## Session Overview

Erik presented a proposal for refactoring the installation and uninstallation flows in the Apsis One Integrations service. The core issue is that the current installation process has overly nested functions with too many responsibilities, making error handling brittle—particularly during uninstallation when external API credentials may have expired. The refactoring aims to "lift" error handling from deeply nested function calls to the top-level orchestrator (the installation flow itself), allowing granular control over which errors to ignore and how to recover. Two major function groups were identified as candidates for refactoring: the bootstrap functions and the install connector function.

---

## Installation Flow Architecture

### Current Multi-Step Installation Process

The installation flow performs several sequential operations:

1. **Keyspace Creation**: Creates a keyspace specific to each CRM system (e.g., `dynamics_keyspace` for Microsoft Dynamics, `fsc_enterprise_keyspace` for FSC Enterprise)

2. **Bootstrap Process**: For CRM systems that request it, predefined attributes and events are created inside Apsis. This is driven by a schema defined for each integration type.

3. **Default Mappings Setup**: Maps external CRM identifiers to Apsis internal fields (e.g., CRM Contact ID maps to Apsis contact field)

4. **Resource Generation**: Generates an A1 API key and sends it to the CRM system along with metadata (section discriminator, section ID, keyspace discriminator, keyspace ID)

5. **Connectivity Registration**: Stores connection details in the database, including API URL and credentials. The specifics vary by CRM system.

> [Erik]: "Probably like the more complex flow of all of our of all of our services... it has the most by far the most steps involved"

---

## The Install Connector Function Problem

### Current Implementation Issues

The `install_connector` function bundles too many responsibilities into a single call. For FSC Enterprise as an example:

1. Takes the API URL and API key entered by the customer in the UI
2. Stores credentials in a database table (e.g., `con_fsc` for FSC connections)
3. Makes a request to the CRM system to generate webhooks
4. Retrieves the webhook ID from the CRM response
5. Stores the webhook ID in the webhooks table with a link back to the connectivity record

**The critical problem**: If any step fails, all subsequent steps are skipped.

### Why Uninstallation Exposes the Issue

During uninstallation, the same flow is attempted in reverse: delete API key/URL from database, delete webhooks from the CRM system, delete webhook records from the database. However, a real-world failure scenario exposes the brittleness:

[Erik]: > "If the API key has been rotated then we will never be able to remove the webhooks from the CRM. But we would still want to remove them from within Apsis."

The current architecture prevents this because:
- The function tries to delete the webhook from the CRM first (using potentially expired/rotated credentials)
- If that request fails, the entire function halts
- The webhook record never gets removed from the Apsis database
- This creates "lingering or trailing webhooks" with no record of what should be cleaned up on future attempts

### The Flag Propagation Problem

Today there is an `ignore_external_errors` flag that allows the system to proceed despite CRM API failures. However, because these responsibilities are nested several function calls deep, the flag must be passed through every intermediate function, creating a maintenance burden.

[Erik]: > "Because these functions are so nested, like you have the install connector which calls one function, which calls one function which calls one function, we would need to pass this flag so deep and so many like it's so many nested requests that it becomes unmanageable after a while."

---

## Bootstrap Functions Refactoring

Currently, a single bootstrap function is responsible for:
- Creating the folder structure for attributes
- Creating attributes
- Creating events

The refactoring proposes splitting this into three separate, explicit calls:
- `bootstrap_attributes()`
- `bootstrap_events()`
- etc.

This allows the main installation orchestrator to handle errors for each step independently, rather than having one "God function" that either succeeds entirely or fails entirely.

---

## Proposed Refactoring Architecture

### Core Principle: Lift Responsibility to the Orchestrator

Instead of:
```
installation_flow()
  → install_connector()
    → store_credentials()
    → create_webhooks()
      → external_api_call()
```

Refactor to:
```
installation_flow()
  → store_credentials()
  → create_webhooks()
  → bootstrap_attributes()
  → bootstrap_events()
  → register_connectivity()
  [Error handling at each step]
```

**Benefit**: The installation flow itself becomes the orchestrator, deciding which errors to ignore and how to proceed. No flag propagation through nested calls needed.

[Erik]: > "You should not call like install connector and inside install connector you make the request. You do it in the actual installation flow because then you can handle all the errors in that main flow. You can choose to ignore those errors without having to pass any flags like 5 steps deeper in the flow."

### Error Handling Philosophy

The refactoring does not prevent incidents, but significantly simplifies error handling:

[Erik]: > "I would not really agree, maybe as it is described, that this solution will prevent incidents... However, it will by far simplify error handling for all of this because you can catch the errors and decide what to do with the error when you call like the create webhooks—do you want to proceed and delete things in the database or not?"

---

## Scope of Refactoring

### Affected Connectors

The installation flow refactoring affects **all connectors** because all of them:
- Store API credentials
- Split credential storage into at least two steps (store credentials, then create webhooks)
- Can fail at each step

This includes:
- Generic Connector
- Microsoft Dynamics
- FSC Enterprise
- Other legacy connectors

### Full Sync Producer Not Affected (For Now)

The full sync producer is not a priority for this refactoring because:
- It is currently stable and clear
- Errors in the full sync flow are easy to locate and diagnose
- The primary pain point is in the installation/uninstallation flow due to external API credential issues

The only difference between generic and legacy connectors (like Dynamics) in the full sync is:
- Generic connector: single step to retrieve data from CRM
- Dynamics: retrieve data, then transform it to match Apsis/Justin formats

[Erik]: > "The full sync, however, is not as affected because the only real difference between the generic connector and the more legacy connectors like Dynamics is like which endpoint in the CRM system are we calling when we retrieve the data."

---

## Implementation Planning

### Identified Stories

From this discussion, at least two major stories were identified:

1. **Bootstrap Functions Refactoring**: Break the monolithic bootstrap function into separate calls for attributes, events, and folder creation
2. **Install Connector Refactoring**: Separate credential storage from webhook creation

Both will require subdividing into smaller tasks (e.g., "check supported features," "create database entry," "generate webhooks").

### Next Steps

Erik will:
1. Create an epic in the issue tracking system with the problem statement
2. Create technical stories for each refactoring task
3. Schedule a grooming session with Michal to review the actual code locations and finalize story breakdowns

[Michal]: > "I think you can create a meeting to groom them together, but at least I would like to have possibility to read them before the meeting and familiarize a bit with [the stories]."

Proposed grooming session: Friday, 11:00-12:00 (1 hour initial slot, with option to extend if needed)

---

## Secondary Observations

### Error Logging Level Changes

Erik mentioned routine maintenance tasks like converting fatal pay-duty errors to warnings. These are typically addressed ad-hoc as they arise and do not require formal grooming. Example: In MA, level 50 errors in CloudWatch trigger pay-duty alarms; in the integration service, errors are logged but alarms are based on log level thresholds.

### Full Sync Producer Uncertainty

Erik has a second item on the grooming agenda related to the full sync producer, but acknowledged he needs to investigate what the requirements actually are before discussing them. He suspects there may be a misunderstanding about whether the proposed changes are already implemented.

[Erik]: > "I honestly don't really understand what this is supposed to mean because like in my opinion, like this is already what we are doing. So I need to check that a little bit."

This item is lower priority than the installation flow refactoring and will likely be addressed after further analysis.

---

## Key Takeaways

1. **The Problem is Real**: Uninstallation fails gracefully when external API credentials expire, leaving dangling webhooks in the database with no way to track them for cleanup.

2. **Root Cause is Architectural**: Overly nested functions with multiple responsibilities make it impossible to handle errors granularly without passing flags through many layers.

3. **The Solution is Straightforward**: Flatten the call hierarchy and let the top-level installation orchestrator make explicit calls to single-responsibility functions, capturing and handling errors at the orchestrator level.

4. **Scope is Broad**: The refactoring affects all connector types (generic and legacy) but not the full sync producer (at least not initially).

5. **Implementation is Planned**: Two main stories are identified (Bootstrap and Install Connector refactoring), with a grooming session scheduled to break them into smaller, implementable tasks.

6. **No Silver Bullet**: While the refactoring will improve maintainability and error handling, it will not automatically prevent incidents—it simply makes error recovery more manageable.

---

## Unresolved Questions

1. **Full Sync Producer Refactoring**: Erik needs to investigate whether the second item on the grooming agenda (related to full sync producer) describes work that is already done or actually needs to be done. Status: pending analysis.

2. **Additional Refactoring Candidates**: Erik mentioned he will review the installation flow again to identify if there are other problematic nested functions beyond the two main stories (Bootstrap and Install Connector).

---

## Action Items

- **Erik**: Create an epic for the installation flow refactoring with problem statement and initial technical stories; prepare code examples for the grooming session
- **Erik**: Investigate and clarify the full sync producer refactoring requirements before the grooming session
- **Erik & Michal**: Schedule and conduct grooming session (Friday 11:00-12:00) to review code, finalize story breakdown, and identify additional refactoring opportunities