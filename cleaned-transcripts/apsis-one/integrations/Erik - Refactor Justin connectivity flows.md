---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One - Integrations
topics: [Installation Flow Refactoring, Connector Setup, Webhook Management, Error Handling, Bootstrap Process, Uninstallation Flow, Credential Management]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Installation Flow, Bootstrap Functions, Install Connector, Uninstall Connector, Webhook Management, CRM API Credentials, Key Spaces, Default Mappings, Full Sync Producer]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson walks through a proposed refactoring of the **installation and connector setup flows** in the Apsis One Integrations domain. The core problem is that the current installation process has deeply nested functions with too much responsibility per function, particularly in the `install_connector` and bootstrap functions. This creates brittle error handling, especially during uninstallations when external API errors occur (e.g., expired API keys). The solution is to flatten the flow by moving orchestration up to the main installation flow level and breaking monolithic functions into smaller, focused calls with explicit error handling at each step.

---

## Current Installation Flow Architecture

### Overview of Installation Steps

The installation process for CRM integrations involves multiple sequential steps:

1. **Key Space Creation**: When installing any CRM system (Microsoft Dynamics, FSC Enterprise, etc.), a corresponding key space is created in Apsis (e.g., `dynamics_key_space`, `fsc_enterprise_key_space`)

2. **Bootstrap Attributes and Events**: Some CRM systems require specific attributes and events to be pre-created in Apsis during installation
   - A predefined schema defines which attributes and events should exist
   - If an integration has such a schema, it is bootstrapped at installation time

3. **Default Mappings Setup**: The system establishes default field mappings between CRM and Apsis
   - Example: Maps the CRM ID field to a configured ID field on contacts/persons (e.g., `Contact ID` for Microsoft Dynamics)

4. **Resource Generation**: Apsis generates resources needed for the integration
   - Generates API keys
   - Sends credentials and metadata to the CRM system including:
     - Section discriminator
     - Section ID
     - Key space discriminator and ID
   - CRM can use these in custom flows

5. **Connector Registration**: Final step registers connectivity details in the database
   - Stores API URL and credentials
   - Details vary by CRM type

### The Install Connector Function Problem

[Erik Andersson]: The current `install_connector` function exhibits the core problem. For example, with FSC Enterprise (generic connector):

**Current Monolithic Approach:**
```
install_connector() {
  - Store API URL and API key in con_FSC table (connections table)
  - Create webhooks in the CRM system via API request
  - Store webhook IDs in webhooks table with link to connectivity record
  - If ANY step fails, entire function fails and stops
}
```

This single function has too many responsibilities bundled together. If the API credential storage succeeds but webhook creation fails, or vice versa, the error handling is all-or-nothing at the function level.

---

## The Uninstallation Problem: Why Nested Functions Fail

### The Uninstallation Challenge

[Erik Andersson]: The uninstallation flow reveals the brittleness most clearly:

> During uninstallation we follow essentially the same flow. We try to delete things in the database, so we try to delete the API key and the URL. And we also try to delete webhooks that we have and we only delete the webhooks in our database after we have made a request to the CRM system to delete the corresponding webhook ID because if we were to delete it from our database, then there will most likely be lingering or trailing webhooks if we were to fail to delete them.

**The Critical Issue:**
- Webhooks must be tracked in the database to support retry scenarios
- If we delete a webhook from the database before successfully deleting it from the CRM, and the CRM deletion fails, we lose the webhook ID and cannot clean it up
- However, if the CRM's API credentials have expired or been rotated, we cannot make requests to the CRM to delete webhooks
- The current nested function structure prevents us from deleting database records independently when external API calls fail

**Current Problem Flow:**
```
uninstall_connector() {
  - Try to get CRM functionality (API call)
    - IF THIS FAILS → entire function stops
  - Try to delete webhooks from CRM (API call)
    - IF THIS FAILS → database records never deleted
  - Try to delete database records
    - Only reached if all above succeeded
}
```

Result: If API credentials are expired, nothing gets cleaned up from the Apsis database, leaving orphaned records.

### The "Ignore External Errors" Flag Limitation

[Erik Andersson]: There is already a flag called `ignore_external_errors` that should allow the flow to proceed despite API failures:

> Because these functions are so nested, like you have the install_connector which calls one function, which calls one function which calls one function, we would need to pass this flag on so deep and so many like it's so many nested requests that it becomes unmanageable after a while.

The flag needs to be threaded through 5+ levels of nested function calls, making the code unmaintainable and error-prone.

---

## The Bootstrap Function Problem

[Erik Andersson]: A similar issue exists with bootstrap functions:

> During the bootstrap, we have one function which is responsible for creating the folder, creating attributes, creating events there should be one call to bootstrap attributes and then there should be one call to bootstrap events and et cetera, et cetera.

The current `bootstrap()` function handles:
- Creating the folder structure
- Creating attributes
- Creating events

If creating attributes fails, events are never created. These should be separate, independently-callable operations.

---

## Proposed Solution: Flattening and Orchestration

### Core Principle

[Erik Andersson]: Move responsibility from nested functions to the main installation flow:

> What this flow aims to do is to lift like remove this nesting and make and make the installation installation flow be like the orchestrator of all of these things. So like in the installation flow you should make a explicit call to say I want to create the web hooks for this CRM system. You should not call install_connector and inside install_connector you make the request.

### New Architecture

Instead of:
```
installation_flow()
  → install_connector()
      → create_webhooks()
          → make_api_call()
```

The refactored version would be:
```
installation_flow()
  → store_credentials()
  → create_webhooks()
  → bootstrap_attributes()
  → bootstrap_events()
  → register_connectivity()
```

**Benefits:**
1. **Explicit Error Handling**: Each call to a top-level function can have its own try-catch and error decision logic
2. **Independent Failure**: If webhook creation fails, you can still delete credentials from the database without passing flags through nested layers
3. **No Flag Threading**: The `ignore_external_errors` decision happens at the orchestration level, not passed through 5+ nested calls
4. **Clearer Code**: The installation flow becomes self-documenting—you see exactly what steps happen and in what order

### Specific Stories Identified

[Michal Rosikiewicz and Erik Andersson identify two major refactoring stories]:

1. **Bootstrap Refactoring**: Split the monolithic bootstrap function into separate calls:
   - `bootstrap_attributes()`
   - `bootstrap_events()`
   - Potentially: `bootstrap_folder()` or similar

2. **Install Connector Refactoring**: Split into focused operations:
   - `store_credentials()` — saves API URL and key to `con_FSC` table (or equivalent)
   - `create_webhooks()` — makes CRM API call to create webhooks and stores IDs
   - Potentially: `check_supported_features()` as a separate step

[Erik Andersson]: Each of these will become separate technical stories within an epic, allowing for clearer implementation and testing.

---

## Impact Assessment

### Scope of Refactoring

[Erik Andersson]: The installation flow affects all connectors:

> The installation flow affects everyone because all of all of the connectors have like 1. Store API credentials like all all of them are split into at least two steps where like we create the web hooks in the second step and things can go wrong in the first one.

This includes:
- Generic connector (supported features logic)
- Legacy connectors (Microsoft Dynamics, FSC Enterprise, etc.)

### Full Sync Producer vs. Installation Flow

[Erik Andersson]: The refactoring should not affect full sync in the same way:

> The full sync, however, is not as affected because the only real difference between the generic connector and the more legacy connectors like Dynamics is like which endpoint in the CRM system are we calling when we retrieve the data.

- Generic connector: retrieves data directly in required format
- Dynamics (legacy): retrieves data, then transforms via `mutate_data_to_match_justin_formats()`

[Erik Andersson notes]: There is a second refactoring story Benjamin mentioned regarding full sync producer, but Erik needs to investigate further as he's not confident he understands its intent:

> I don't really understand what this is supposed to mean because like in my opinion, like this is already what we are doing.

### Priority Assessment

[Erik Andersson]: Installation flow refactoring is higher priority than full sync refactoring:

> This is this is not definitely not as urgent as the installation flow because the installation flow we already now know that there are problems on uninstallation due to like expired API credentials, whereas the full sync producer is rather stable and quite clear.

The installation flow has known failure modes; the full sync is stable.

---

## Implementation Planning

### Grooming Meeting Approach

[Michal Rosikiewicz and Erik Andersson agree on process]:

- Erik will create an epic and identify the component problems
- Erik will draft the technical stories in advance
- Michal will review the stories before a grooming meeting to familiarize himself
- They will meet to groom the stories together, with Erik showing the code locations where these issues exist
- Estimated time: 1 to 1.5 hours

[Erik Andersson]: 

> I can create the epic and the problem and then we create the technical stories. Like we groom the actual stories inside that epic.

### Related Cleanup Work

[Erik Andersson mentions lower-priority work that doesn't require grooming]:

> Apart from this, of course there are always going to be like stories like change this paid duty error into a warning instead. But I don't think that requires that much grooming because that is literally like change log level to warning.

These smaller issues can be handled as they arise. However, Erik notes he uses different log level handling in different services:

**Integration service**: Triggers PagerDuty alarms on log level "error" in CloudWatch

**MA service**: Triggers PagerDuty on log level 50 (numeric log level)

[Michal Rosikiewicz indicates]: His services also use log levels to differentiate errors and warnings in code, and could potentially align with level 50 in the future.

---

## Caveats and Clarifications

### Will This Prevent Incidents?

[Erik Andersson]: This refactoring should not be oversold:

> I would not really agree, maybe as it is described, that this solution will prevent incidents that would be a bit taking it a bit too far maybe. However, it will by far simplify error handling for all of this because you can catch the errors and decide what to do with the error when you call like the create web hooks.

The refactoring improves maintainability and error handling clarity, not incident prevention per se.

### Full Sync Producer: Unresolved Understanding

[Erik Andersson] indicates he needs to investigate the second refactoring story regarding the full sync producer:

> I I need to analyze that a bit more. That will probably not be for Friday.

This suggests Benjamin recommended refactoring the full sync producer flow as well, but Erik doesn't yet understand what changes are intended. This is not an urgent blocker.

---

## Key Takeaways

1. **Current Problem**: The installation and uninstallation flows have deeply nested functions where multiple operations are bundled together, making error handling brittle and impossible to recover from certain failure modes (e.g., expired credentials during uninstall).

2. **Root Cause**: Functions like `install_connector()` and `bootstrap()` handle too many responsibilities. If any internal step fails, all dependent steps are blocked, and the `ignore_external_errors` flag cannot be propagated through nested call chains.

3. **Solution**: Refactor to an orchestrator pattern where the main `installation_flow()` makes explicit, sequential calls to focused functions (`store_credentials()`, `create_webhooks()`, `bootstrap_attributes()`, etc.). Error handling happens at the orchestration level, not buried in nested functions.

4. **Key Benefit for Uninstallation**: When external APIs fail (e.g., expired credentials), the flow can still clean up Apsis database records independently, preventing orphaned data.

5. **Scope**: Affects all CRM connectors (generic and legacy). The full sync producer is less affected and less urgent.

6. **Next Steps**: 
   - Create epic and draft stories (Erik)
   - Share stories for pre-review (Erik → Michal)
   - Schedule grooming meeting (~1–1.5 hours) to review code and finalize stories
   - Investigate the second refactoring story about full sync producer (Erik)

---

## Unresolved Questions

1. **Full Sync Producer Refactoring**: What exactly is Benjamin's proposed refactoring for the full sync producer? Erik needs to investigate and confirm his understanding before this can be groomed.

2. **Story Breakdown**: Beyond `store_credentials()` and `create_webhooks()` for the connector, are there other independent operations that should be split out (e.g., `check_supported_features()` as a separate story)?

3. **Backward Compatibility**: Are there any concerns about changing the installation flow signature or behavior that might affect existing installations or integrations?