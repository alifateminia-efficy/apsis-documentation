---
source_file: Integration installation story grooming.txt
domain: Apsis One - Integrations
topics: [Installation/Uninstallation Flow Refactoring, Error Handling in Integration Manager, Webhook Management, Connector Interface Design, Code Complexity Reduction, Database Transaction Handling]
speakers: [Erik Andersson, Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Integration Manager, Installer Interface, Connector Types (FSC Enterprise, Dynamics, Lime, Generic), Webhook Registration/Unregistration, Credential Storage, Bootstrap Logic, Operations Bus]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covers the refactoring of the **Apsis One Integrations** installation and uninstallation flow. The core problem addressed is excessive nesting and responsibility consolidation in connector uninstall functions, which makes error handling brittle and difficult to maintain. The discussion focuses on breaking apart monolithic connector functions into smaller, more granular operations (webhook registration/unregistration, credential storage/removal, bootstrap operations) that can be called directly from the Integration Manager, eliminating the need to pass control flags through multiple levels of nested function calls. The team sketches out an epic and associated stories to implement this refactoring incrementally across all supported connectors.

---

## Current Installation/Uninstallation Flow Architecture

### Problem Statement: Over-Nested Functions with Complex Error Handling

[Erik Andersson]: The core issue is that installation and uninstallation functions today have far too many responsibilities. They are nested four to five levels deep, requiring extensive parameter passing, and the error handling becomes extremely cumbersome.

The prime example is webhook removal during uninstallation. Consider this scenario:

1. **Integration Manager** calls `uninstall` on the specific connector (e.g., FSC Enterprise)
2. **Connector uninstall function** attempts to unregister webhooks from the CRM system
3. If the customer has already rotated their API credentials (intentionally or by accident), the CRM unregister call fails with `401` or `403`
4. **The function stops execution** — the database cleanup for webhook subscription IDs never happens
5. Result: **leftover webhook data in the database**, creating a problematic state

### The Operations Bus Account Delete Scenario

[Erik Andersson]: This problem is particularly acute when the Operations Bus triggers account deletion. In that flow:

- The system attempts to uninstall integrations for a deleted account
- External CRM systems may be unreachable or the API credentials may be invalid
- The desired behavior is **best-effort cleanup**: remove everything on our side regardless of CRM failures
- Current design forces the database cleanup to be skipped if the CRM call fails

This violates an important security principle: we should not leave webhook subscriptions pointing to a deleted account in the external CRM, as the customer might continue sending us private data.

### Current Workaround: Flag Passing Pattern

The current mitigation uses control flags (`ignore_external_system_errors`) that are passed through the entire call stack:

```
Integration Manager
  └─ uninstall()
      └─ FSC Enterprise uninstall(ignore_external_system_errors)
          └─ unregister_webhooks(ignore_external_system_errors)
              └─ delete_webhook_database_entries(ignore_external_system_errors)
```

This approach becomes increasingly difficult to maintain as nesting deepens and as new flags are needed.

---

## Proposed Refactoring: Breaking Apart Monolithic Connector Functions

### Core Redesign Philosophy

[Erik Andersson]: Instead of having one large connector-specific function (`FSC_Enterprise.install()`, `FSC_Enterprise.uninstall()`) that handles all bootstrap, webhook registration, and credential management, we should break these into discrete operations that are orchestrated at the **Integration Manager level**.

This achieves several benefits:

1. **Explicit control flow** — The Integration Manager can clearly state: "Now we're creating the folder, now bootstrapping attributes, now registering webhooks, now deleting webhooks"
2. **Unified error handling** — Errors are handled in one place rather than threaded through nested function calls
3. **Conditional execution** — The `ignore_external_system_errors` flag can be checked once at the top level rather than passed down through the stack
4. **Parallelizable operations** — Some operations could potentially run in parallel (e.g., unregistering webhooks for multiple integrations)

### New Connector Interface Design

The **Installer Interface** (currently in `libraries/connectors/types`) defines what operations each connector must support. The refactoring introduces new granular functions:

**For webhook management:**
```
- register_webhooks()
- unregister_webhooks()
```

**For credential management:**
```
- store_credentials()
- remove_credentials()
```

**For bootstrap operations:**
```
- create_folder()
- bootstrap_attributes()
- bootstrap_events()
```

Instead of:
```
- install()
- uninstall()
- bootstrap()
```

### Error Handling Strategy During Uninstallation

[Erik Andersson]: The uninstallation logic needs to be aware of two scenarios:

1. **Normal user-initiated uninstall** (default behavior):
   - If any step fails (e.g., webhook unregister fails because API key is invalid), **abort the entire flow**
   - Do NOT delete database entries if we can't clean up the CRM
   - This prevents orphaned webhook subscriptions in the CRM that could leak customer data

2. **Operations Bus account deletion** (with `ignore_external_system_errors=true`):
   - If webhook unregister fails, **proceed anyway**
   - Clean up all database entries regardless
   - Rationale: The account is being deleted; we must remove our records

[Erik Andersson]: The customer support use case also leverages this mechanism. If a customer accidentally rotates their API key and cannot uninstall:
- Support can call the integration manager endpoint directly with `ignore_external_system_errors=true`
- This allows the customer to force-remove the integration from their end, even if the CRM side fails

---

## Detailed Installation Flow Components

### Step-by-Step Installation Process

[Erik Andersson]: The installation flow today includes these sequential steps:

1. **Create custom key space** — Database setup for this integration instance
2. **Create delegation key** — For API authentication
3. **Create API key** — Credentials for CRM access
4. **Bootstrap custom attributes** — Ensure all custom attributes exist in CRM that we need for syncing
5. **Bootstrap events** — Ensure all event listeners are configured in CRM
6. **Whitelist attributes** — Configure which attributes are allowed for sync
7. **Register webhooks** — Set up CRM callbacks to notify us of changes

### Uninstallation Flow

1. **Initialize database transaction** — Wrap everything for atomicity
2. **Remove event listeners for consents** — Stop listening to consent changes
3. **Delete mappings** — Remove attribute mapping configurations
4. **Delete sync conditions** — Remove sync rule configurations
5. **Disable campaign syncing** — Remove listeners for form submit events, email events, etc.
6. **Unregister webhooks** (CRM API call) — Tell CRM to stop sending callbacks
7. **Delete webhook database entries** — Clean up webhook subscription IDs stored locally
8. **Delete credentials** (database operation) — Remove API key and other auth data

---

## Connector Implementation Complexity

### Introducing the Installer Interface Pattern

[Erik Andersson]: Each connector (FSC Enterprise, Dynamics CRM, Lime CRM, Generic connector) implements a set of **interfaces** that define required operations.

Located in: `libraries/connectors/types`

Example: **FSC Enterprise Installer Interface**

Currently, the interface requires:
- `install()` — Handle full installation
- `uninstall()` — Handle full uninstallation
- (Optionally) `bootstrap()` — Handle bootstrapping

If a connector doesn't support a specific operation, it can return `nil`.

### Current FSC Enterprise Uninstall Implementation

Within the FSC Enterprise connector uninstall:

```
function uninstall():
    1. Call unregister_webhooks()
       └─ Makes HTTP request to CRM API to remove webhook subscriptions
       └─ Also deletes webhook subscription IDs from local database
    2. Call SQL DELETE to remove credentials from database
```

**The problem**: If step 1 fails (API key rotated), step 2 never executes.

### Why Adding Functions to the Interface is Costly

[Erik Andersson]: Adding a new required function to the Installer Interface seems simple — it takes 3 minutes. However:

- **Every connector that supports that operation must implement it**
- For example, adding `register_webhooks()` requires implementation in:
  - FSC Enterprise
  - Dynamics
  - Lime
  - Generic Connector
- This spans across all integration connectors, turning a 3-minute interface change into **approximately 2 days of implementation work** across all connectors

This is why the refactoring is proposed as an **epic with incremental stories**.

---

## Proposed Story Breakdown and Estimation

### Refactoring Strategy: Start with FSC Enterprise

[Erik Andersson]: The recommendation is to start with FSC Enterprise as a reference implementation:

1. Add the granular functions to the Installer Interface
2. Implement these functions specifically for FSC Enterprise
3. This gives the team a complete example to follow for other connectors
4. Subsequent implementations can then proceed in parallel per-connector

### Story 1: Add Webhook Registration to Installer Interface
- **Effort estimate**: 5 story points (collaborative effort across team members)
- **Breakdown**:
  - Add `register_webhooks()` function signature to interface: ~2-3 days per team member learning the code
  - Implement for FSC Enterprise: ~2-3 days
  - Implement for Dynamics: ~2-3 days
  - Implement for Lime: ~2-3 days
  - Implement for Generic connector: ~2-3 days

[Michal Rosikiewicz]: The team can split responsibilities — one person handles registration, another handles deletion, allowing parallel work on different connectors.

[Erik Andersson]: If each team member implements the logic for 1-2 connectors, they will become familiar with the pattern and can then take on remaining connectors more quickly.

### Story 2: Add Webhook Unregistration to Installer Interface
- **Effort estimate**: 5 story points
- Same breakdown as registration
- **Critical complexity**: This story must properly handle the `ignore_external_system_errors` flag during unregistration

[Erik Andersson]: This is more important than registration because webhook unregistration failures are what cause database inconsistencies and leftover data.

### Story 3: Add Credential Storage Functions to Installer Interface
- **Effort estimate**: 5 story points
- Functions: `store_credentials()` and `remove_credentials()`
- Must be implemented per-connector since credential storage is connector-specific

### Story 4: Refactor Bootstrap Logic (Combine Attributes and Events)
- **Effort estimate**: 2 story points
- **Scope**: Consolidate bootstrap operations into separate functions
- Functions: `create_folder()`, `bootstrap_attributes()`, `bootstrap_events()`
- **Why smaller estimate**: This logic is **not connector-specific** and is centralized
- **Lower priority**: Bootstrap failures have not caused as many operational issues as webhook-related failures

[Michal Rosikiewicz]: Combine the attributes and events bootstrap into a single story since the refactoring logic is essentially identical.

### Integration Manager Orchestration Refactoring
- **Not estimated as a separate story** — this is done as part of implementing the above stories
- The Integration Manager will be updated to call the new granular functions in sequence

---

## Risk and Considerations

### Backward Compatibility and Staged Rollout

[Erik Andersson]: Because changes to the Installer Interface are mandatory for all connectors:

1. A branch can be created with the interface changes
2. The interface is added but implementations are stubbed/not yet enforced
3. Different team members can work on different connectors in parallel
4. Each connector implementation can be tested independently

### Customer Support Patterns

[Erik Andersson]: The refactoring supports an existing customer support flow:

**Scenario**: Customer accidentally rotates API key and cannot uninstall.

**Current workaround**: Support team calls the Integration Manager endpoint with `ignore_external_system_errors=true`:

```
POST /integration-manager/force-uninstall
{
  "integration_id": "...",
  "ignore_external_system_errors": true
}
```

This pattern will continue to work post-refactoring because error handling happens at the Integration Manager level rather than buried in nested connector functions.

[Erik Andersson]: I will provide Postman request examples in a collection for support team reference.

### Estimation Rationale: Why Team Member Estimates Differ from Original Author

[Erik Andersson]: I estimate 2 days to implement a new function across all connectors (because I'm the original author and know the codebase deeply). However, for team members new to the connectors:

- First connector: ~2-3 days (learning curve)
- Subsequent connectors: ~1-2 days each (pattern now familiar)

[Michal Rosikiewicz]: A reasonable team estimate should account for learning: suggest doubling the original author's estimate for team members.

[Erik Andersson]: This follows the established estimation pattern in the team: estimates for maintenance work done by Erik are typically doubled when other team members implement similar work.

---

## Process and Timeline

### Current Development Capacity

[Lukasz Grabowski]: There is currently no planned product work for the Integrations domain. This is a good window for the team to work on technical improvements.

[Erik Andersson]: This is the benefit — the team can take on these smaller KT exercises together without disrupting product delivery.

### Estimation for Board Visibility

[Lukasz Grabowski]: Even though the team has flexibility, estimates are needed for board visualization:

- **Purpose**: Show product team that integration refactoring is progressing
- **Granularity**: Should estimate at the story level, not the subtask level
- **Approach**: List subtasks in the story description; aggregate story points across all subtask implementations

### Execution Plan

[Erik Andersson]: Suggests dedicating focused time (e.g., one week in December) to working on these stories together:

1. One person owns the interface changes (adds new function signatures)
2. Team members pair/collaborate on reference implementations (FSC Enterprise)
3. Once comfortable, team members can work on different connectors in parallel
4. Erik will review implementations and provide feedback
5. Remaining technical refinements will be captured in additional stories and added to an epic

[Lukasz Grabowski]: The real value is in the team getting hands-on experience with the codebase, even if all stories aren't completed:

> "The purpose is just touch the code right and implement 1-2 tasks and then you can review it and check, so we can feel more comfortable with the code that we did something in the code and it worked and etcetera."

---

## Key Takeaways

1. **The Core Problem**: Current installation/uninstallation flows have functions nested 4-5 levels deep with error handling that breaks when external systems (CRM APIs) fail. This creates database inconsistencies and security risks (orphaned webhooks).

2. **The Solution**: Break monolithic connector functions into granular operations (`register_webhooks`, `unregister_webhooks`, `store_credentials`, `remove_credentials`, etc.) that are orchestrated at the Integration Manager level. This centralizes error handling and eliminates deep parameter passing.

3. **Interface-Driven Design**: The Installer Interface pattern requires all connectors to implement new functions. Adding a single function appears simple but requires implementation across 4+ connectors (~2 days of work per function).

4. **Error Handling Strategy**: 
   - Default: Abort uninstallation if any step fails (prevent orphaned CRM webhooks)
   - Operations Bus: `ignore_external_system_errors` flag allows best-effort cleanup on account deletion
   - Support flow: Allow forced uninstall via direct Integration Manager endpoint

5. **Estimation Approach**: Use story-level estimates that account for learning curves. Team member estimates should be approximately 2x the original author's estimates. Subtasks can be listed in description but don't require individual estimates.

6. **Staged Rollout**: Start with FSC Enterprise as a reference implementation. Once team is comfortable with the pattern, parallel work on other connectors is feasible.

7. **Business Value**: Improves maintainability, reduces customer support burden (force-uninstall capability), and reduces operational incidents from database inconsistencies.

---

## Unresolved Questions and Action Items

1. **Epic Creation**: Erik Andersson will polish story descriptions and create an overarching epic encompassing all refactoring stories.

2. **Additional Technical Stories**: Erik mentioned creating "some technical stories that will also need to be done" but these were not fully defined by end of session. These will be added to the epic once finalized.

3. **Postman Collection**: Erik committed to providing Postman request examples for the `force-uninstall` endpoint pattern used by customer support.

4. **Timeline Confirmation**: Suggested December for focused KT work, but final timeline depends on product priorities and team capacity.

5. **Priority Ordering**: Within the epic, webhook stories (registration/unregistration) have higher priority than bootstrap stories due to operational impact.