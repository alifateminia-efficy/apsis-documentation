---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One Integrations
topics: [Installation Flow Refactoring, Error Handling Architecture, Nested Function Decomposition, Connector Setup, Webhook Management, Bootstrap Process, Uninstallation Flows]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Installation Flow, Bootstrap Process, Connector Functions, Webhook Management, Database Connectivity Storage, API Credentials Management, Full Sync Producer]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration]
---

## Session Overview

Erik presents a case for refactoring the installation flow for CRM connectors to improve error handling and maintainability. The current flow bundles too many responsibilities into single functions, making it difficult to handle errors gracefully—particularly during uninstallation when external systems may be unreachable. The proposed solution is to decompose nested functions into explicit, separate calls at the orchestrator level (the main installation flow), allowing errors to be caught and handled independently. Michal agrees with the approach and discusses how to break down two key problem areas: the bootstrap functions and the install connector function.

---

## Current Installation Flow Architecture

### What Happens During Installation

The installation process is the most complex flow across all Apsis One services. It has multiple sequential steps that must coordinate between Apsis and external CRM systems.

**Key steps include:**

1. **Keyspace Creation**: A dedicated keyspace is created for each CRM system type
   - For Microsoft Dynamics: `dynamics` keyspace
   - For FSC Enterprise: `fsc_enterprise` keyspace
   - Pattern applies to all supported CRM systems

2. **Bootstrap Schema Setup**: Some CRM systems require predefined attributes and events to exist in Apsis
   - Apsis maintains predefined schemas for each supported integration
   - If an integration has a schema, it is bootstrapped during installation
   - This includes creating attributes and events in Apsis

3. **Default Mappings**: CRM identifiers are mapped to Apsis fields
   - Example: CRM ID mapped to a configured ID field on contacts/persons
   - For Microsoft Dynamics: Contact ID mapping
   - These mappings ensure data consistency between systems

4. **Resource Generation and Credential Sharing**: Resources created in Apsis are sent to the CRM
   - A1 API key is generated
   - Section discriminator and ID are included
   - Keyspace discriminator and ID are sent
   - CRM system uses this information for custom flows and integration

5. **Connectivity Registration**: Final step stores connection details in the database
   - API URL and credentials are stored
   - Configuration varies by CRM system type

---

## The Problem: Monolithic Function Design

### The Install Connector Function Issue

The `install_connector` function (used for generic connectors like FSC Enterprise) combines multiple distinct responsibilities:

1. **Store Credentials in Database**
   - Takes API URL and API key from customer input
   - Stores in the connections table (e.g., `con_fsc` table for FSC)
   - Creates a database entry linking to this connectivity

2. **Create Webhooks in External CRM**
   - Makes API request to CRM system asking it to generate webhooks
   - Receives webhook ID back from CRM
   - Stores webhook ID in webhooks table with foreign key to connectivity entry

**The Problem**: All of this happens in a single function. If any step fails, all subsequent steps are skipped. This is problematic but tolerable during installation (failures abort cleanly). During uninstallation, it becomes critical.

### Uninstallation Scenario: Why This Matters

[Erik Andersson]: The core issue manifests during uninstallation:

During uninstallation, the flow attempts to reverse the installation process:
1. Delete API key and URL from database
2. Delete webhooks by making API request to CRM system
3. Only delete webhook entries from Apsis database AFTER successful CRM deletion

**The Critical Problem**: The flow is designed to only proceed if all previous steps succeed. However, during uninstallation:
- API credentials may have expired or been rotated by the customer
- The CRM system may be unreachable or may return external errors
- If the API key fetch fails, we cannot make requests to the CRM to delete webhooks
- If we delete webhook database entries without successfully deleting them from the CRM, we lose the ability to clean them up in future attempts
- We would have orphaned/trailing webhooks in the external CRM system

**Current Workaround**: There is an `ignore_external_errors` flag that allows the flow to proceed despite errors. However, because these functions are deeply nested, passing this flag through the call chain becomes unmanageable.

[Erik Andersson]: 
> "Because these functions are so nested, like you have the install connector which calls one function, which calls one function which calls one function, we would need to pass this flag on so deep and so many like it's so many nested requests that it becomes unmanageable after a while."

---

## The Bootstrap Function Problem

Similar issues exist with bootstrap functions. Currently, one function is responsible for:
- Creating the folder
- Creating attributes
- Creating events

These are distinct operations that should have independent error handling.

---

## Proposed Solution: Orchestrator-Level Refactoring

### Core Principle

Move error handling and orchestration from deeply nested functions to the main installation flow. Make the installation flow the explicit orchestrator of all operations.

### Specific Changes

**For Install Connector**: Instead of calling `install_connector()` which does everything:
```
1. Call: store_credentials(api_url, api_key)
   - Handles credentials storage independently
   - Catches and reports errors at orchestrator level

2. Call: create_webhooks(connectivity_id)
   - Handles webhook creation independently
   - Catches and reports errors at orchestrator level
```

**For Bootstrap**: Instead of one bootstrap function:
```
1. Call: bootstrap_attributes()
2. Call: bootstrap_events()
3. (Additional bootstrap calls as needed)
```

Each call is independent and errors are handled at the orchestrator level.

### Benefits

[Erik Andersson]:
> "This flow aims to do is to lift, remove this nesting and make the installation flow be like the orchestrator of all of these things... then you can handle all the errors in that main flow. You can choose to ignore those errors without having to pass any flags like 5 steps deeper in the flow."

**Error Handling Improvements**:
- Errors are caught at the point of call, not buried in nested functions
- Decision logic (proceed vs. abort) stays at the orchestrator level
- Flags like `ignore_external_errors` can be applied locally instead of threaded through the call chain
- Uninstallation can selectively ignore external errors while still cleaning up database state

**Code Clarity**:
- Each function has a single, clear responsibility
- Call sequence is explicit and visible in the installation flow
- New developers can understand the flow by reading the orchestrator function
- Easier to add logging and monitoring at each step

### Scope and Impact

**Affects All Connectors**: The installation flow impacts all connector types because all store API credentials and create webhooks in at least two separate steps.

**Does Not Affect Full Sync Producer (Yet)**: The full sync producer is currently stable. The only differences between generic and legacy connectors (like Dynamics) are:
- Different endpoint calls to retrieve data from CRM
- For Dynamics specifically: an extra transformation step (mutate data to match Justin format)
  - Generic connector: calls endpoint → data is in required format
  - Dynamics: calls endpoint → transforms/mutates data → returns in required format
- The current approach already handles this, so the full sync producer is not urgent for refactoring

---

## Implementation Plan

### Stories to Be Created

1. **Bootstrap Refactoring Story**: Split bootstrap function into separate calls for attributes, events, etc.

2. **Install Connector Refactoring Story**: Split into distinct calls:
   - Store credentials
   - Create webhooks
   - Potentially separate calls for checking supported features

3. **Additional Stories**: Erik notes there may be other opportunities for decomposition that will be identified during code review

### Execution Approach

Erik proposes creating an epic with the problem statement, then creating technical stories within it. The stories will be groomed together.

[Michal Rosikiewicz]: 
> "I think you can create a meeting to groom them together, but at least I would like to have possibility to read them before the meeting and familiarize a bit with."

**Proposed Process**:
1. Erik creates the epic and problem statements
2. Erik creates initial technical stories
3. Joint meeting to groom stories in detail (1 to 1.5 hours estimated)
4. Erik will show code examples during grooming to provide context
5. Stories will be worked on for future pair programming sessions (both for implementation and educational purposes)

### Schedule

- Meeting scheduled for Friday, 11 AM – 12 PM (with possibility to extend if needed)
- Michal will read the stories in advance to familiarize himself

---

## Important Caveats and Clarifications

### Will This Prevent Incidents?

[Erik Andersson]:
> "I would not really agree, maybe as it is described, that this solution will prevent incidents that would be a bit taking it a bit too far maybe. However, it will by far simplify error handling for all of this..."

The refactoring improves maintainability and error handling capability but is not positioned as a complete incident prevention measure.

### Full Sync Producer Uncertainty

[Erik Andersson]: 
> "The second one, I have actually no idea what this is about. I'm gonna need to check what this might actually be... I need to analyze that a bit more. That will probably not be for Friday."

Erik needs to review Benjamin's recommendation regarding the full sync producer more thoroughly. Current assessment: it's stable and not urgent compared to the installation flow work.

---

## Related Technical Decisions

### Pay Duty Error Handling

Erik mentions that there are straightforward stories around changing "pay duty errors" to warnings. These don't require deep grooming since they are simple log-level changes:
- In MA service: Triggers pay duty alarms on log level "error" (CloudWatch level 50)
- In Michal's services: Uses lock levels defined in code for warnings and errors
- Erik notes: In Integration service, such changes won't happen in the foreseeable future unless deliberately changed

---

## Key Takeaways

1. **Current Problem**: The installation and uninstallation flows have monolithic functions with multiple responsibilities bundled together, making error handling during uninstallation particularly problematic when external systems are unreachable.

2. **Root Cause**: Deeply nested function calls require error-handling flags to be passed many levels deep, which is unmaintainable.

3. **Solution**: Decompose nested functions into explicit, independent calls at the orchestrator level (main installation flow), enabling local error handling decisions without flag threading.

4. **Priority**: Installation flow refactoring is urgent and affects all connectors. Full sync producer refactoring requires further analysis and is lower priority.

5. **Next Step**: Create epic, stories, and schedule grooming session for Friday 11 AM – 12 PM. Michal will pre-read stories before the meeting.

6. **Scope**: Two major stories initially (Bootstrap, Install Connector), with potential for additional stories identified during code review.

---

## Unresolved Questions and Action Items

### Action Items

- **Erik**: Create epic and technical stories for Bootstrap refactoring and Install Connector refactoring
- **Erik**: Review Benjamin's recommendation on full sync producer changes and determine scope/urgency
- **Erik**: Prepare code examples for grooming session showing current nested structure
- **Michal**: Read created stories before Friday grooming session
- **Both**: Schedule and conduct grooming session Friday 11 AM – 12 PM

### Open Questions

1. **Full Sync Producer Changes**: What exactly is Benjamin's recommendation? Is it in scope for this refactoring effort?
2. **Additional Decomposition Opportunities**: Are there other functions beyond Bootstrap and Install Connector that would benefit from this refactoring?