---
source_file: Integration installation story grooming.txt
domain: Apsis One Integrations
topics: [Installation and Uninstallation Flow Refactoring, Error Handling in CRM Integrations, Webhook Management, Connector Interface Design, Code Architecture Improvements]
speakers: [Erik Andersson, Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Integration Manager, Connector Interfaces, FSC Enterprise Connector, Dynamics Connector, Lime Connector, Generic Connector, Webhook Registration/Unregistration, Credentials Storage, Bootstrap Logic]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session focused on identifying and planning a major refactoring effort for the **installation and uninstallation flow** in the Apsis One Integrations system. The core problem identified is that current connector uninstall functions have excessive nesting (4-5 levels deep), cumbersome error handling, and mixed responsibilities that make it difficult to handle scenarios where external CRM system errors should be ignored (particularly during account deletion operations). The team discussed breaking down monolithic connector functions into smaller, single-responsibility functions that can be called directly from the Integration Manager, improving code clarity and error handling capability. Multiple user stories were created to guide this refactoring work, with the team estimating the effort involved.

---

## The Core Problem: Complex Nested Installation/Uninstallation Flows

### Current Architecture Issues

[Erik Andersson]: The fundamental issue is that installation and uninstallation functions in connectors have way too much responsibility. They are deeply nested (4-5 levels deep) and require passing arguments through multiple layers, making error handling extremely cumbersome.

### The Webhook Removal Error Scenario

A concrete example illustrates the problem:

> When a customer wants to remove webhooks during uninstallation, if the API credentials have been rotated in the CRM system (the customer deleted our API key), the unregister webhooks request fails with a 401 or 403 error. Because the error is encountered, the function stops execution immediately. The database entries for webhook subscription IDs are never deleted, leaving orphaned data in the system.

This is particularly problematic for **account deletion operations via the operations bus**. In this scenario, we want to perform a "best effort" cleanup: even if the CRM system rejects our requests (because the account is already deleted or credentials are invalid), we should still remove all webhook data and credentials from our database.

### The Parameter-Passing Problem

To support the `ignore_external_system_errors` flag (which allows the flow to continue despite CRM failures), the flag must currently be passed through every nested function layer:

```
Integration Manager -> uninstall() 
  -> connector.uninstall(ignore_external_system_errors)
    -> connector.unregister_webhooks(ignore_external_system_errors)
      -> connector.delete_webhook_db_entries(ignore_external_system_errors)
```

This creates a brittle, hard-to-maintain code structure.

---

## Proposed Solution: Flat, Single-Responsibility Functions

### The Refactoring Goal

Instead of having one monolithic `install()` and `uninstall()` function per connector that orchestrates everything, break the flow into smaller, focused functions that the **Integration Manager calls directly**. Error handling happens at the Integration Manager level, eliminating the need to pass flags through nested function calls.

### New Connector Interface Functions

[Erik Andersson]: The connector interface should expose smaller, composable functions:

**For registration/installation:**
- `register_webhooks()`
- `store_credentials()`
- `create_folder()` (bootstrap)
- `bootstrap_attributes()`
- `bootstrap_events()`

**For unregistration/uninstallation:**
- `unregister_webhooks()`
- `remove_credentials()`

### Updated Integration Manager Flow

Instead of:
```
installer.uninstall(ignore_external_system_errors)
```

The Integration Manager directly orchestrates:
```
// Explicit, sequential flow
integration_manager.remove_event_listeners()
integration_manager.delete_mappings()
integration_manager.delete_sync_conditions()
integration_manager.disable_campaign_syncing()

// Then call connector-specific functions directly
connector.unregister_webhooks()  // Handle errors here if needed
connector.remove_credentials()   // Handle errors here if needed
```

**Benefit**: Error handling is centralized. The `ignore_external_system_errors` flag is checked at the Integration Manager level. If unregister_webhooks() fails due to a 401 and the flag is set, we catch the error, log it, and proceed to remove_credentials() anyway.

---

## Implementation Strategy: Gradual Refactoring

### Starting Point: FSC Enterprise

[Erik Andersson]: We should start with FSC Enterprise, which is a simpler connector. Once we add the new functions to the connector interface, we need to implement them in **every connector** that currently supports that operation.

The effort is significant because:
- Adding a function to the interface takes ~3 minutes
- Implementing the logic across all connectors (Dynamics, Enterprise, Lime, Generic Connector, etc.) takes ~2 days

[Erik Andersson]: If you do it iteratively and one of you works through one connector while I work on another, we can parallelize the work.

### Estimated Effort

The team discussed estimation:

- **Register Webhooks story** (implement across all connectors): **5 story points**
  - ~2-3 days per developer per connector
  - Can be split: one developer handles installation, another handles uninstallation across different connectors
  
- **Unregister Webhooks story**: Included in above estimate (same complexity)

- **Store/Remove Credentials story**: **Similar to Register Webhooks** (~5 story points)

- **Bootstrap Attributes and Bootstrap Events** (combine into one story): **2 story points**
  - This only happens in one place (Integration Manager)
  - Considerably smaller refactoring than connector-specific functions

[Michal Rosikiewicz]: Since you're all senior developers, the team settled on **4-5 story points** as reasonable estimates, acknowledging that actual time might vary depending on connector complexity and unforeseen issues.

---

## Important Context: The `ignore_external_system_errors` Flag

### Why This Flag Exists

[Erik Andersson]: By default, the uninstallation process aborts completely if any error is encountered. This is intentional for information security reasons: if we remove our database records while webhooks remain registered in the CRM, the customer might continue sending us private data even after they believe they've uninstalled the integration.

However, there are two scenarios where we **must ignore external system errors**:

1. **Account deletion via operations bus**: When an account is terminated in Apsis, we call a special endpoint that sets the flag to ignore external errors. We perform best-effort cleanup regardless of CRM system state.

2. **Customer-initiated forced removal**: If a customer accidentally rotated their API key and cannot reinstall because uninstall is now blocked, they can request a **force removal** via direct API call to the Integration Manager, which overrides the default behavior.

[Erik Andersson]: We have Postman requests available for these scenarios, which will be shared with the team.

---

## Story Creation and Tracking

The team created stories in their issue tracking system (appears to be Jira-like, with parent/child relationships). Discussion points about story management:

### Subtasks vs. Story Point Estimates

[Lukasz Grabowski]: Stories should have descriptions that list what needs to be done. If there are subtasks (e.g., "implement register_webhooks for Dynamics," "implement register_webhooks for Enterprise"), these should be listed in the description or as subtasks.

[Michal Rosikiewicz]: Story point estimates sum up the work across all implementations. For example, a "Register Webhooks" story worth 5 points might break down as:
- Add to interface: trivial
- Enterprise implementation: 2 points (dev effort)
- Dynamics implementation: 2 points (dev effort)
- Lime implementation: 1 point (dev effort)

### Parallel Work Approach

[Michal Rosikiewicz]: The team can split work by connector or by phase. For example, one developer tackles the installation path across all connectors while another tackles the deletion path. This avoids merge conflicts and allows parallel progress.

---

## Installation vs. Uninstallation: Different Error Handling Semantics

[Michal Rosikiewicz]: An important caveat: installation can fail for the same reasons (CRM system refusing requests). However, during installation, we **should not** use the ignore errors flag by default. If installation fails, we want to abort and retry.

[Erik Andersson]: Exactly. The difference is:
- **Installation failure**: Abort and fail. Don't leave partial state.
- **Uninstallation failure**: Depends on context.
  - Normal case: Abort and fail (don't remove our records if CRM still has webhooks)
  - Account deletion case: Best effort cleanup (remove our records regardless)

---

## Connector-Specific Implementation Notes

### Why Complexity Varies by Connector

[Erik Andersson]: Each connector has different CRM APIs, different authentication schemes, and different webhook mechanisms. While the pattern is the same (register webhooks, store credentials, etc.), the actual implementation details are unique to each CRM system.

Connectors currently in scope:
- FSC Enterprise (Efficy Enterprise 12.0/12.1)
- Microsoft Dynamics
- Lime
- Generic Connector
- Intermale (mentioned in estimation discussion)

### Starting Lean

The team chose to start with FSC Enterprise as the pilot to establish the pattern before rolling it out to more complex connectors.

---

## Scheduling and Team Capacity

### No Planned Work Currently

[Lukasz Grabowski]: There is currently no committed product work for the integrations team, which provides flexibility to pick up these technical improvement tasks.

[Erik Andersson]: This is actually beneficial. The team can work on these refactoring efforts ad-hoc, using the available capacity to improve code quality without competing priorities.

### Process Change

[Lukasz Grabowski]: The team recently consolidated into a single team structure, which eliminates the need for big-room planning ceremonies. Work can be more dynamically prioritized.

---

## Next Steps

[Erik Andersson]: 
- Stories have been created for the major refactoring efforts
- Additional technical stories will be created to cover refinements and fixes
- These will all be grouped under an epic in the issue tracker
- The team will start implementation when product priorities allow, likely in December
- Erik will polish the story descriptions and provide additional technical context

[Erik Andersson]: If implementation is blocked or questions arise, he can assist with code reviews and guidance to help the team move through connectors more efficiently.

---

## Key Takeaways

1. **The core problem is solved by splitting responsibilities**: Move webhook registration/unregistration and credentials storage out of monolithic connector functions and into smaller, focused functions called directly from the Integration Manager.

2. **Error handling becomes declarative**: Instead of threading an `ignore_external_system_errors` flag through nested functions, it's checked once at the Integration Manager level where orchestration happens.

3. **The refactoring is large but parallelizable**: Adding a function to the interface is trivial; implementing it across connectors takes time. However, work can be split by connector or by operation (install vs. uninstall) to enable parallel development.

4. **Two distinct error semantics exist**: Installation should fail fast; uninstallation should support a "best effort cleanup" mode when called from account deletion operations.

5. **Bootstrap logic is separate and simpler**: Creating folders and bootstrapping attributes/events don't require the same level of refactoring and can be tackled independently with a smaller story point estimate.

6. **Team capacity supports this work**: No competing product roadmap items currently exist, providing a window to improve integration code quality.

---

## Unresolved Questions & Action Items

- **Polish story descriptions**: Erik will refine story text and add any technical details needed for implementation
- **Technical stories**: Erik committed to creating additional technical stories covering refinements and fixes beyond the main refactoring
- **Postman collection**: Customer-facing API requests (force remove, etc.) will be shared in a collection for testing
- **Implementation schedule**: Dependent on product priorities; no firm date set, likely December or after