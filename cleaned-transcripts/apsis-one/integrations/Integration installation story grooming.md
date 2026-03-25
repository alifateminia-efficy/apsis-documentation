---
source_file: Integration installation story grooming.txt
domain: Apsis One Integrations
topics: [Installation Flow Refactoring, Error Handling Strategy, Webhook Management, Connector Interface Design, Code Simplification, Story Estimation]
speakers: [Erik Andersson, Tomasz Kowalski, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Integration Manager, Connector Interfaces, FSC Enterprise Connector, Dynamics Connector, Lime Connector, Generic Connector, Webhook Registration/Unregistration, Credentials Storage, Bootstrap Logic]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session focused on grooming and designing a major refactoring epic for the Apsis One integration installation flow. Erik Andersson, the original architect, walked the team through the current problems with the installation/uninstallation process: deeply nested functions with poor separation of concerns, cumbersome parameter passing (especially for error-handling flags), and inflexible error recovery strategies. The session resulted in the creation of multiple user stories to incrementally improve the architecture by breaking down large connector functions into smaller, composable operations that can be called directly from the Integration Manager.

---

## Current Architecture Problems

### Nested Functions and Parameter Proliferation

The current installation flow has functions nested 4-5 levels deep with excessive parameter passing. A concrete example is the uninstallation process:

> "In the installation flow today, a lot of the functions have way too much responsibilities. They are like we have a lot of functions that are nested four to five steps. You need to pass on the arguments a lot, a lot, and above all, the error handling becomes very clunky."

[Erik Andersson]: When calling the specific uninstall function on a connector (e.g., `dynamics connector.uninstall()` or `enterprise connector.uninstall()`), multiple control flow flags need to be threaded through every nested function call. For instance, if an `ignore_external_system_errors` flag needs to be respected at the leaf level, it must be passed down through every intermediate function.

### The API Key Rotation Problem

A critical real-world scenario illustrates the design flaw:

[Erik Andersson]: Consider the FSC Enterprise uninstall flow. The current `uninstall()` function does three things:
1. Calls the CRM system to unregister webhooks
2. Deletes webhook subscription IDs from the database
3. Deletes the stored API credentials from the database

If a customer has rotated their API key in the CRM system before uninstalling the integration, the unregister call will fail with a 401 or 403 error. Since the function is aborted on error, the code never reaches the database cleanup steps. This leaves orphaned webhook records in the Apsis database.

> "If you encounter an error during this uninstallation for, say, enterprise and the API key might have been rotated already. That means that we fail to unregister the web hooks because we are not authorized to make any unregister calls to the CRM. Then that will result in an error, which means that the removal of the database entries for us for the webhook data like that will not happen because we encountered an error before."

### The Operations Bus Account Delete Scenario

When the operations bus triggers an account deletion, Apsis must clean up all local state even if the CRM is unreachable:

[Erik Andersson]: The system needs to support two error-handling modes:
- **Default (strict):** Abort uninstallation if any external call fails. This prevents orphaned webhooks that could receive customer data after integration termination—a critical security concern.
- **Force clean (lenient):** Used when operations bus deletes an account. Apsis performs best-effort cleanup locally, ignoring any CRM system errors.

> "The operations bus for account deletes? Because then we want to essentially ignore all errors from the CRM system and remove everything we have, even if we did fail to remove the web hooks in the CRM, because otherwise we will continuously send a failure request to you, which we agreed on that we shouldn't. We should do a best effort and clean that up."

Currently, this is controlled by passing an `ignore_external_system_errors` flag through the entire call stack, which is unmaintainable as the number of connectors grows.

---

## Proposed Refactoring Strategy

### High-Level Vision

Instead of having one large, multi-responsibility `install()` or `uninstall()` function per connector, the Integration Manager should directly orchestrate smaller, single-purpose operations. This makes the control flow explicit and error handling locally contained.

[Erik Andersson]: The goal is to move from this pattern:
```
Integration Manager
  ↓ calls
Connector.uninstall(ignore_errors_flag)
  ↓ calls (passing flag)
Connector.unregister_webhooks(ignore_errors_flag)
  ↓ calls (passing flag)
Database.delete_webhook_subscriptions(ignore_errors_flag)
```

To this pattern:
```
Integration Manager:
  1. Call connector.unregister_webhooks()
  2. Catch errors locally
  3. If ignore_errors_flag, proceed anyway
  4. Call connector.delete_stored_credentials()
  5. Handle errors at this level
```

> "The integration manager should make the request to create the webhooks. And the integration manager should make the request to delete the web hooks. Then you can handle this ignore errors like you don't need to pass that along. You can just accept like you can handle all of this inside the Integration Manager. Also it will be a lot easier to follow this the flow."

### Benefits

1. **Explicit flow:** Reading the Integration Manager code shows exactly what happens during installation/uninstallation in order.
2. **Localized error handling:** Error recovery logic lives at the point where the decision is made, not buried in callback functions.
3. **Parallelization opportunities:** Individual operations can potentially be executed in parallel (e.g., unregistering webhooks and deleting credentials simultaneously).
4. **Testability:** Smaller functions are easier to unit test with various error scenarios.

---

## Proposed Connector Interface Changes

### New Installer Interface Methods

The refactoring introduces new, granular interface methods that all connectors must implement:

**For webhook management:**
- `register_webhooks()` — called during installation to register webhooks in the CRM
- `unregister_webhooks()` — called during uninstallation; must respect the `ignore_external_errors` flag

**For credentials:**
- `store_credentials(config)` — persist API keys and connection details to the Apsis database
- `remove_credentials()` — delete stored credentials from the database

**For bootstrap operations:**
- `create_folder()` — create Apsis's custom folder structure in the CRM if it doesn't exist
- `bootstrap_attributes()` — create or sync standard attributes; no longer bundled with other bootstrap logic
- `bootstrap_events()` — create or sync standard events; separate from attributes

[Erik Andersson]: Currently, a single `bootstrap()` method on the connector handles creating the folder, bootstrapping attributes, and bootstrapping events all at once. Splitting these allows the Integration Manager to call them independently and handle errors more precisely.

### Current Connector Implementations

The system currently supports four primary connectors, each with varying complexity:

- **FSC Enterprise** — Relatively straightforward; good candidate for refactoring exercise
- **Dynamics** (Microsoft Dynamics) — More complex webhook management
- **Lime** — Similar complexity to Dynamics
- **Generic Connector** — Used as a fallback; less commonly used but still requires updates

[Erik Andersson]: When you add a new method to the installer interface, **every connector must implement it**, even if the implementation is simply `return nil`. Adding a single interface method is trivial (3 minutes), but implementing the logic across four connectors takes approximately 2 days for an experienced developer.

---

## Proposed User Stories

The refactoring is broken into discrete stories that can be worked on independently or in parallel. This approach allows team members to become comfortable with the codebase incrementally.

### Story 1: Add `register_webhooks()` to Installer Interface
**Estimate:** 5 story points (accounting for implementation across all connectors)

**Description:** Introduce a `register_webhooks()` method to the installer interface. During this story:
- Add the method signature to the installer interface
- Implement `register_webhooks()` logic for FSC Enterprise as an example
- Implement the same logic for Dynamics, Lime, and Generic Connector

[Erik Andersson]: For experienced developers on this team, implementing the logic for one connector should take approximately 2-3 days. With two team members, one could handle Enterprise + Lime while another does Dynamics + Generic. However, the effort varies depending on familiarity with each connector's CRM API.

[Michal Rosikiewicz]: These could potentially be parallelized — one developer implements the registration path, another implements the deletion path, reducing total elapsed time.

### Story 2: Add `unregister_webhooks()` with Error Handling
**Estimate:** 5 story points

**Description:** Implement `unregister_webhooks()` for all connectors, with particular attention to the `ignore_external_system_errors` flag.

**Sub-tasks:**
- Unregister webhook logic for FSC Enterprise
- Unregister webhook logic for Dynamics
- Unregister webhook logic for Lime
- Unregister webhook logic for Generic Connector

[Erik Andersson]: This story is critical because it's where external API failures occur. The method must handle the case where the CRM returns an authorization error (e.g., rotated API key) and either abort cleanly or proceed based on the flag:

```
unregister_webhooks(ignore_external_errors: bool):
  try:
    api.unregister(webhook_ids)
  catch error:
    if not ignore_external_errors:
      throw error
    # else: log and continue
  database.delete_webhook_subscriptions()  # Always happens
```

### Story 3: Add `store_credentials()` and `remove_credentials()`
**Estimate:** 2 story points

**Description:** Extract credential management from the installer's bootstrap and uninstall methods.

[Erik Andersson]: This is simpler than the webhook stories because it doesn't vary significantly per connector. The logic is mostly the same across all connectors: serialize the API key and endpoint, store it in the database, delete it on removal.

### Story 4: Extract Bootstrap Operations (`create_folder()`, `bootstrap_attributes()`, `bootstrap_events()`)
**Estimate:** 2-3 story points (combined)

**Description:** Split the current monolithic `bootstrap()` method into three separate interface methods.

[Michal Rosikiewicz]: [Suggestion] These two bootstrap operations (`bootstrap_attributes()` and `bootstrap_events()`) are conceptually similar and could be combined into a single story.

[Erik Andersson]: That's reasonable. The combined story would be smaller than the webhook ones because bootstrap logic is less sensitive to error handling; unlike webhook registration, bootstrap operations are idempotent and don't interact with external systems in ways that can cause cascading failures.

### Story 5: Refactor Integration Manager to Use New Interface Methods
**Estimate:** 3 story points

**Description:** Update the main installation and uninstallation flows in the Integration Manager to:
1. Call `register_webhooks()` directly instead of `connector.install()`
2. Call `unregister_webhooks()` with proper error handling
3. Orchestrate `store_credentials()`, `bootstrap_*()`, etc. with explicit ordering
4. Handle the `ignore_external_system_errors` flag at the Integration Manager level, not threaded through connectors

[Erik Andersson]: This is where the refactoring pays off. The control flow becomes a simple script: do X, handle errors, do Y, handle errors, etc.

---

## Error Handling Philosophy

### Default Behavior: Abort on Error

By default, uninstallation should fail if any external system call fails:

[Erik Andersson]: This is intentional. If we remove local state (webhook subscriptions from our database) before successfully unregistering in the CRM, the customer's CRM will continue sending data to a webhook endpoint that no longer exists. From a security perspective, this is unacceptable if the endpoint contains personal information.

> "We don't want to remove things from our database if there are left over things in the CRM system because that can lead to some issues if there are trailing web hooks and the customer continues sending us like private data when they have technically stopped using the integration. From an information security perspective, this is potentially very, very bad."

### Force-Clean Mode: Best-Effort Cleanup

Two scenarios trigger force-clean mode:

1. **Operations bus account deletion:** When a customer's entire account is deleted, Apsis must clean up all local records regardless of CRM availability.
2. **Customer-initiated force uninstall:** If a customer has rotated API credentials and cannot uninstall normally, they can contact support to force-remove the integration.

[Erik Andersson]: The operations team can call the Integration Manager with a flag to override normal error handling:

> "Like you can make a direct request of course to the integration manager and override this default behaviour to say that no like I I will force remove this integrations. So we handle occasionally request from customers where they want to reinstall but they can't reinstall because they can't uninstall because something they accidentally did."

This is implemented via a direct API call to the Integration Manager with `force_ignore_external_errors=true`, allowing support teams to manually reset customer installations.

---

## Implementation Approach and Timeline

### Iterative Rollout

The team decided against implementing all stories at once. Instead:

[Erik Andersson]: Start with FSC Enterprise as the pilot. Implement the new interface methods for Enterprise, understand the pattern, then apply it to the other connectors. This allows the team to build familiarity gradually.

[Lukasz Grabowski]: We'll introduce these stories into the backlog alongside regular product work. Since there are no other planned initiatives for the integrations team right now, these refactoring tasks can be picked up opportunistically.

> "The benefit right now is that there is no planned work for integration. So we can essentially come up with things as we come. We can take these smaller exercises and work on them together like but we we we almost essentially have no real backlog to speak about apart from technical improvements and refinements."

### Estimation Rationale

[Erik Andersson]: I (Erik) can implement these changes faster because I understand the original architecture. For team members new to the code, the same logic might take 2-3 times longer. Estimates reflect the **team's** effort, not Erik's.

[Michal Rosikiewicz]: We'll double the estimate for implementation by team members who aren't as familiar with the code.

[Erik Andersson]: That's a fair heuristic we've used before. If I estimate 1 day, the team should budget 2 days.

### Why Not Estimate Now?

[Lukasz Grabowski]: Initially, the team questioned whether estimation was necessary given the ad-hoc nature of the work. However, estimation serves a practical purpose: it helps visualize work on the board and makes clear trade-offs between technical work and product features.

> "I think in in this terms it's important to know how big is it or at least some tasks, right? Because it's not like we need to implement everything you we we have you. But it's the the purpose is just touch the code right and implement 1-2 tasks and then you can review it and check and guys for we we can feel better come more comfortable with the code."

The goal is **not** to predict exactly how long stories will take, but to:
- Make the work visible on the board
- Give product stakeholders a sense of technical investment
- Help team members gain confidence by shipping complete, reviewable changes

---

## Code Structure Reference

### Connector Interface Definition

```
libraries/connectors/types/
  installer.interface.ts (or equivalent)
    - install() function signature
    - uninstall() function signature
    - [NEW] register_webhooks()
    - [NEW] unregister_webhooks()
    - [NEW] store_credentials()
    - [NEW] remove_credentials()
    - [NEW] bootstrap_*() functions
```

### Connector Implementations

Each connector lives in a separate package:
- `connectors/fsc-enterprise/`
- `connectors/dynamics/`
- `connectors/lime/`
- `connectors/generic/`

### Integration Manager

```
services/integration-manager/
  install.flow.ts
  uninstall.flow.ts
```

Currently calls high-level `connector.install()` and `connector.uninstall()`. Post-refactoring, these will orchestrate smaller operations directly.

---

## Post-Implementation Technical Work

[Erik Andersson]: Beyond the stories above, additional technical refinements are needed. I will create separate stories for these and summarize them in an epic. Examples include:

- Refactoring how credentials are encrypted/stored (currently mixed into connector logic)
- Improving webhook retry logic when registration fails transiently
- Adding observability/logging to track which steps of installation/uninstallation succeed or fail
- Database schema changes to support better state tracking during partial failures

These will be refinement and polish work that makes the refactored code production-ready.

---

## Key Takeaways

1. **Current architecture conflates concerns:** Connector install/uninstall methods mix webhook management, credential storage, and bootstrap logic, making error handling inflexible and code hard to follow.

2. **Security-first error handling:** The system must default to aborting uninstallation on error to prevent orphaned webhooks. Force-clean mode is reserved for account deletion and customer support scenarios.

3. **Interface-driven refactoring:** By introducing small, focused interface methods, the code becomes testable, maintainable, and allows the Integration Manager to have explicit control flow.

4. **Incremental approach:** Rather than refactoring all connectors at once, start with FSC Enterprise. Once the pattern is established and team members are comfortable, parallelize work across other connectors.

5. **Estimation for visibility, not precision:** Stories should be estimated to make technical work visible on the board and to help team members gauge progress, not to predict exact hours. Expect estimates to be adjusted as implementation proceeds.

6. **Parallelization opportunities:** Once the interface is defined, different team members can work on different connectors in parallel, reducing total elapsed time.

---

## Unresolved Questions and Action Items

- **Erik Andersson** to polish the drafted stories (add acceptance criteria, link to relevant code locations) before they're ready for development.
- **Erik Andersson** to create additional technical refinement stories related to credential management, retry logic, and observability.
- **Team** to prioritize these stories relative to product roadmap once product team provides input.
- **Team** to schedule a follow-up session once the first story (register_webhooks) is being worked on, to verify understanding and discuss any implementation surprises.
- **Clarification needed:** Whether the `ignore_external_system_errors` flag should be a parameter to individual connector methods or handled at the Integration Manager level (current understanding: Integration Manager level, but implementation should verify this works for all connectors).