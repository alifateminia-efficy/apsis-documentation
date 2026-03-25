---
source_file: Integration installation story grooming.txt
domain: Apsis One Integrations
topics: [Installation Flow Refactoring, Error Handling in Integration Lifecycle, Webhook Management, Connector Interface Design, Code Complexity Reduction]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Integration Manager, Connector Interfaces, FSC Enterprise Connector, Dynamics Connector, Lime Connector, Generic Connector, Webhook Registration/Unregistration, API Credentials Management, Bootstrap Logic]
session_type: architecture-review
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration]
---

## Session Overview

This session focused on grooming user stories for a major refactoring epic aimed at simplifying the installation and uninstallation flows in Apsis One Integrations. The core problem is that connector installation functions have excessive responsibilities with deep nesting (4-5 levels), making error handling cumbersome and difficult to maintain. The proposed solution breaks down monolithic connector functions into smaller, focused operations called directly from the Integration Manager, improving code clarity and error handling. The team discussed the scope, technical approach, and effort estimation for implementing these changes across multiple CRM connectors.

---

## Problem Statement: Current Installation Flow Architecture

### The Core Issue with Nested Functions

Erik Andersson identified a fundamental architectural problem in how the installation flow is currently structured:

> The big part of this is that in the installation flow today, a lot of the functions have way too much responsibilities. They are like we have a lot of functions that are nested four to five steps. You need to pass on the arguments a lot, a lot, and above all, the error handling becomes very clunky.

The primary pain point is **excessive parameter passing through nested function calls**. When control flow features like error handling flags need to propagate through multiple function layers, maintainability suffers dramatically.

### Real-World Example: Webhook Unregistration Failure

The team uses a concrete scenario to illustrate the problem:

When a customer has already rotated their API credentials in the CRM system and then attempts to uninstall the integration:

1. The `uninstall` function in the Integration Manager calls the connector-specific `uninstall` function (e.g., for FSC Enterprise)
2. That function attempts to call `unregister_webhooks` on the CRM system
3. The API call fails with a 401 or 403 error (unauthorized)
4. Because the function stops at this error, **the code never reaches the database cleanup step** that removes webhook subscription IDs and API credential records
5. Result: leftover data remains in the database, creating a broken state

This becomes a critical issue during **account deletion operations from the Operations Bus**, where the team has agreed that:

> We want to essentially ignore all errors from the CRM system and remove everything we have, even if we did fail to remove the webhooks in the CRM, because otherwise we will continuously send a failure request to you, which we agreed on that we shouldn't. We should do a best effort and clean that up.

### The Flag Propagation Problem

To handle this requirement, the team introduced an `ignore_external_system_errors` flag that should allow the flow to continue despite CRM failures. However:

> The issue here is that this flag of course needs to be continuously passed down through every nested function that we have. So we set the flag in the uninstall function in the integration manager and then we call the uninstallation function for the enterprise connector in that uninstallation. We need to send in the flag and then inside the uninstallation in enterprise we have like two more nested functions and again we need to pass in this flag to each different steps and this becomes very very cumbersome to maintain that for the error handling.

---

## Current Implementation Details

### Connector Interface Pattern

Apsis One uses an interface-based architecture where each connector implements required functions. The `installer` interface defines functions that each connector must support:

```
connectors/types/
├── installer (interface)
├── fsc_enterprise_installer
├── dynamics_installer
├── lime_installer
└── generic_installer
```

Each connector's `installer` interface provides:
- `install()` - Called when user clicks "Install [Connector Name]"
- `uninstall()` - Called when user clicks "Uninstall [Connector Name]"

If a connector doesn't support a specific function, it can return `nil`.

### Uninstall Function Example: FSC Enterprise

The current implementation shows all operations bundled into one function:

1. **Unregister webhooks** - Makes a request to the CRM system to remove registered webhooks
2. **Delete webhook data** - Removes webhook subscription IDs from the Apsis database
3. **Delete credentials** - Removes the API key and related authentication data from the database

All three steps happen in a single function with no separation of concerns. If step 1 fails, steps 2 and 3 never execute.

### Installation Flow Process

During installation, the flow performs these operations in sequence:

1. Create custom key space
2. Create delegation key
3. Generate one-time API key
4. Bootstrap custom attributes (if not already present)
5. Enable attribute whitelisting
6. Register webhooks on the CRM system
7. Store credentials in the database

---

## Proposed Solution: Decomposed Functions

### High-Level Design Change

Instead of having one monolithic `FSC_Enterprise.install()` function, the team proposes moving control flow to the Integration Manager level and having the connector implement focused, single-responsibility functions:

**New connector interface functions:**
- `register_webhooks()` - Only registers webhooks on CRM
- `unregister_webhooks()` - Only unregisters webhooks on CRM
- `store_credentials()` - Only stores credentials in database
- `remove_credentials()` - Only removes credentials from database
- `create_folder()` - Only creates custom folder (for bootstrap)
- `bootstrap_attributes()` - Only bootstraps attributes
- `bootstrap_events()` - Only bootstraps events

### Installation Flow (New Design)

The Integration Manager would orchestrate the flow explicitly:

```
integration_manager.install():
  - create_folder()
  - bootstrap_attributes()
  - bootstrap_events()
  - register_webhooks()  ← Can handle errors here directly
  - store_credentials()  ← Can handle errors here directly
```

Instead of:

```
integration_manager.install():
  - connector.install()  ← All logic hidden, all error handling nested
```

### Error Handling Improvement

With this design, error handling becomes **localized to where decisions are made**:

```
if ignore_external_system_errors:
  try:
    register_webhooks()
  except CRMError:
    log_warning("Failed to register webhooks, continuing")
else:
  # Will raise exception and abort flow
  register_webhooks()
```

The flag does not need to be passed down through nested function calls.

### Benefits of the Refactored Approach

[Erik Andersson]: "You don't have one God function which handles everything related to the installation for the connector. You can see more explicit like now we are doing the bootstrap for attributes, now we're creating the folder, bootstrapping the events we are doing. Registering the webhooks, we are deleting the webhooks, yada, yada, yada."

Key advantages:
- **Explicit flow visibility** - Reading the Integration Manager code tells you exactly what happens
- **Localized error handling** - Errors are caught where the decision to proceed/abort is made
- **Testability** - Each function can be tested independently
- **Reusability** - Functions can be called in different combinations
- **Parallelization** - Functions could potentially be executed in parallel in the future

---

## Implementation Strategy

### Phased Approach

Rather than refactoring all connectors simultaneously, the team proposes:

1. **Update the connector interface** - Add the new function signatures to the interface definition
2. **Implement for one connector** (FSC Enterprise as the pilot) - Serve as the reference implementation
3. **Replicate for remaining connectors** - Dynamics, Lime, Generic Connector, and others

This allows team members to become familiar with the pattern through the first implementation, then apply it in parallel.

### Why FSC Enterprise as Pilot

[Erik Andersson]: "The existing logic for FSC Enterprise looked quite good. So like we would add the installer interface and then we do this exercise for FSC Enterprise. And I'm sure we just here we do this exercise for FSC Enterprise and then we can do the similar one for all of the others."

FSC Enterprise is relatively straightforward and serves as good scaffolding for understanding the refactoring process before tackling more complex connectors.

### Scope Considerations

Adding any function to the connector interface becomes a **mandatory implementation across all connectors**:

> As soon as you change the installer interface here, that means that you are now adding a mandatory function to all of these connectors. So if you add the register webhook function to the interface, now we would need to add the register webhook functions like to the dynamics to enterprise to lime and to the generic connector like every every connector where we currently register webhooks, so it will take 3 minutes to like add the function to the interface. It will most likely take like 2 days to add the logic to each of the connectors.

---

## Special Cases and Requirements

### Operations Bus Account Deletion Flow

When an account is deleted via the Operations Bus, a special flag is set that changes the expected behavior:

> When the customer clicks on uninstall like it, it might fail and we will abort the whole flow. But if you call us on the operations bus then we set the flag to ignore these errors and then we will proceed no matter what.

This flow is critical because:
- If webhooks remain registered after we've deleted our database records, the customer's CRM will continue sending us their data
- From a data security perspective, this is unacceptable - we shouldn't receive private data after the customer has terminated the integration

**Resolution behavior**: The `ignore_external_system_errors` flag ensures we clean up our side (database) even if the CRM is unreachable.

### Customer-Initiated Force Uninstall

The team also handles cases where customers accidentally rotate API keys and can't uninstall normally:

> Let's say that some customer would come to you and say like, oh, hello, I I accidentally forgot my or I mistakenly rotated the API key in the CRM system and I would like to remove the installation. In the exact same fashion where we set the flag to ignore external system errors on the operations bus. Like you can make a direct request of course to the integration manager and override this default behaviour to say that no like I will force remove this integrations.

**Implementation**: Postman requests are provided to customers/support for forcing uninstallation by calling the Integration Manager with `ignore_external_system_errors=true`.

---

## User Stories and Effort Estimation

### Story 1: Add `register_webhooks()` to Connector Interface

**Scope**: Add the function signature to the installer interface. All existing webhook registration logic will eventually move here from the monolithic `install()` function.

**Subtasks** (implementation per connector):
- Implement for FSC Enterprise
- Implement for Dynamics
- Implement for Lime
- Implement for Generic Connector
- Implement for any other connectors

**Effort Estimation**:
- Interface update: negligible (minutes)
- Per-connector implementation: 2-3 days for team members (Erik estimates 2 days for himself, team estimates 3 days for them to account for unfamiliarity)
- **Total story estimate: 5 story points** (representing ~3-4 days of effort considering split work)

> I would imagine this would take me approximately 2 two days to do and depending on how we would want to do it, we could split the responsibilities down. So like if we add, we create a branch for this where we have added the like we have added it to the interface so it is available. If you do that then you it is like very easy to do every connector in parallel like each one of us could sit and play around with one connector each if you would.

The team can work in parallel once the interface is updated - one person per connector.

### Story 2: Add `unregister_webhooks()` to Connector Interface

**Scope**: Similar to Story 1 but for unregistration. This is a higher priority because unregistration failures have caused issues in production.

**Special consideration**: This function **must respect the `ignore_external_system_errors` flag**:

> We need to think about the flags. So the connectors installer function must take the external system into consideration. And but here here on this function where we want to unregister the web hooks this this is where it is very important that we take this flag into consideration. So if we in the unregister flow encounter an error with still remove it from. We still do the clean up in our in the database.

**Effort Estimation**: 5 story points (same reasoning as Story 1)

### Story 3: Add `store_credentials()` to Connector Interface

**Scope**: Extract credential storage logic from the monolithic function.

**Effort Estimation**: 5 story points (parallel work across connectors)

### Story 4: Add `remove_credentials()` to Connector Interface

**Scope**: Extract credential removal logic from the monolithic function.

**Effort Estimation**: 5 story points (parallel work across connectors)

### Story 5: Refactor Bootstrap Logic (Combined)

**Scope**: Instead of a single `bootstrap()` call that handles folder creation, attribute bootstrapping, and event bootstrapping, split into:
- `create_folder()`
- `bootstrap_attributes()`
- `bootstrap_events()`

**Important distinction**: These functions do NOT need to be added to all connectors - the logic is centralized and doesn't vary by connector type.

> This one is not as difficult because this does not need to happen in each of the connectors. This is just done in one place. So this is a considerably smaller refactoring because this logic here this is not tied to the connectors, it doesn't behave differently depending on um if you install Enterprise or Dynamics.

**Effort Estimation**: 2 story points (significantly smaller - only Integration Manager-level changes)

### Story 6: Refactor Installation Flow Integration Manager

**Scope**: Update Integration Manager to call individual connector functions instead of monolithic `install()` function.

**Effort Estimation**: Included in Stories 1-4 implementation

### Priority Discussion

[Erik Andersson]: "Whereas these in particular the in the unregistered, the unregistered flows have caused issues. [The bootstrapping ones are] absolutely not as important. They have not caused that many issues."

**Recommended priority order**:
1. Unregister webhooks (highest priority - has caused issues)
2. Register webhooks
3. Credentials management (store/remove)
4. Bootstrap logic (lower priority - fewer issues)

---

## Estimation Methodology and Discussion

### Why Estimation is Needed

[Lukasz Grabowski]: "Because I think it will be important when for the decision, right, because we will need to visual visualise this on the board for example. OK, so we have product things to do, but yeah, let's do integration as a key T So we spent for example two days on integration, KT implementation. So I think in in this terms it's important to know how big is it or at least some tasks, right?"

Estimates serve multiple purposes:
- **Resource planning** - Deciding how much time to dedicate to KT vs. product work
- **Board visibility** - Tracking progress on refactoring vs. feature work
- **Confidence building** - Team members implementing unfamiliar code need to see completion of working examples

> The purpose is just touch the code right and implement 1-2 tasks and then you can review it and check and guys for we we can feel better come more comfortable with the code that we we did something in the code and it worked.

### Handling Uncertainty

The team discussed that estimates for work done by unfamiliar developers tend to be higher:

> It's typically how how we have handled it. Like if I am doing MA stuff for example, then we have doubled it than if Prem or do is doing it. So I think that's that's pretty fair.

For Stories 1-4, the team settled on **5 story points each** as a reasonable estimate that:
- Accounts for learning curve
- Allows for parallel work across connectors
- Provides visibility for scheduling decisions

### Subtask vs. Story Point Estimates

Discussion around whether to estimate subtasks individually or only at story level:

> What we do usually is that we put like what needs to be done if there are subtasks and on a story level we put story point estimate that sums up. So you can write down subtasks in description.

The team's standard practice is to:
- List subtasks (per-connector implementations) in the story description
- Put a total story point estimate at the story level
- Not create separate story cards for each connector

This allows parallel work while maintaining single tracking unit.

---

## Team and Timeline Context

### Current Capacity and Priorities

[Lukasz Grabowski]: "There is no such plans. I mean, we are waiting for the priorities from, you know, from product."

The integration domain currently has:
- No committed product roadmap items
- Availability to work on technical improvements
- Flexibility to dedicate time to KT implementation

### Organizational Change

The team mentioned an upcoming process change:

> Big room planning planning anymore. We'll change. Yeah, we will change the development and the process... Because you are one team. Yeah, of course.

This affects how the team will schedule work going forward - big room planning meetings will no longer apply to the integrated team structure.

### How Work Will Be Conducted

[Erik Andersson]: "If I can finish this story first. And create a. And on this function. Then maybe we'll need to make the stores a bit more descriptive, but we have you here. So just that the only purpose, but we will see it. I think it depends on the priorities from the product."

The plan:
1. Polish the stories (Erik will refine descriptions and acceptance criteria)
2. Add additional technical stories for refinements and fixes
3. Group stories into an epic
4. Schedule work based on product priorities and capacity
5. Implement with team collaboration, with Erik available for pairing/review

---

## Key Takeaways

1. **The Problem**: Installation/uninstallation functions are deeply nested (4-5 levels) with tightly coupled error handling and parameter passing that makes maintenance difficult.

2. **The Solution**: Decompose monolithic connector functions into single-responsibility functions called directly from the Integration Manager, localizing error handling and improving code readability.

3. **Critical Requirement**: The `ignore_external_system_errors` flag must work correctly for the account deletion flow (Operations Bus), where we must clean up our database even if the CRM is unreachable.

4. **Implementation Approach**: 
   - Update connector interface to define new functions
   - Implement for FSC Enterprise first (pilot/reference)
   - Work in parallel on other connectors once pattern is established
   - Prioritize unregister/register webhook functions (production issues)
   - Lower priority on bootstrap logic (fewer issues)

5. **Effort**: Estimated 5 story points per major refactoring story, accounting for learning curve across team. Bootstrap refactoring is smaller (2 points) because it doesn't vary by connector.

6. **Timeline**: Flexible - work will begin once product priorities are determined. Team capacity exists for this work alongside any product commitments.

7. **Process**: Stories will be polished by Erik and added to an epic. Team will implement with pairing/review support from Erik. Subtasks will be listed in story descriptions for parallel work tracking.

---

## Unresolved Questions and Next Steps

1. **Final Story Polish** - Erik will refine all story descriptions, acceptance criteria, and examples before adding to epic

2. **Technical Stories** - Erik mentioned creating additional technical stories for refinements and fixes alongside the main refactoring stories (to be added to epic)

3. **Product Prioritization** - Work is contingent on product priorities being communicated to the team

4. **Postman Collection** - Erik mentioned providing Postman requests for the force-uninstall functionality that customers/support can use (to be shared separately)

5. **Detailed Implementation Guide** - Once FSC Enterprise is implemented, Erik will review the approach with team to establish pattern for remaining connectors