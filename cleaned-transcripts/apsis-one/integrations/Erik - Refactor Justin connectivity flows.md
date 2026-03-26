---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One Integrations
topics: [Installation flow refactoring, Connector setup process, Error handling in integration flows, Webhook management, Bootstrap functionality, API credential storage, Uninstallation resilience]
speakers: [Erik Andersson, Michal Rosikiewicz]
key_components: [Installation flow, Install connector function, Bootstrap functions, Webhook management, Credential storage, CRM connectivity, Generic connector, Legacy connectors (Dynamics), API key management, Database persistence]
session_type: architecture-review
subdomains: ["Different Types of Connectors"]
---

## Session Overview

Erik presented a proposal to refactor the installation flow for CRM connector integrations, particularly focusing on reducing nested function complexity and improving error handling during both installation and uninstallation. The core issue is that the current monolithic `install_connector` and `uninstall_connector` functions bundle multiple responsibilities together, making error propagation difficult and preventing graceful cleanup when external API calls fail. The proposed solution is to flatten the function hierarchy so the main installation orchestrator explicitly calls granular functions (create credentials, create webhooks, bootstrap attributes, etc.), allowing fine-grained error handling at each step.

---

## Current Installation Flow Architecture

### High-Level Steps

The installation process for a CRM system involves several distinct operations:

1. **Keyspace creation**: A keyspace is created for the specific CRM system (e.g., `dynamics_keyspace` for Microsoft Dynamics, `fsc_enterprise_keyspace` for FSC Enterprise)
2. **Bootstrap phase**: Predefined schema (attributes and events) are created in Apsis One if the integration defines them
3. **Default mapping setup**: CRM IDs are mapped to Apsis One fields (e.g., `Contact ID` for Microsoft Dynamics)
4. **Resource generation**: An A1 API key is generated and sent to the CRM system along with section information (discriminator, ID, keyspace ID, etc.)
5. **Connectivity registration**: API URLs and credentials are stored in the database for later use

### The `install_connector` Function Problem

[Erik Andersson]:

> "The biggest culprit here would be the install connector part. So say for example that you are installing FSC Enterprise... first of all like you have entered the API URL and a API key, so we add a entry in our connections table... In the same function, when this is done, we proceed to create webhooks in the CRM system and we make a request to the CRM, ask them to generate the webhook and then we take the ID of that webhook and we store that in the webhooks table..."

Currently, the `install_connector` function handles:
- Storing the API URL and API key in the connections table (e.g., `con_FSC`)
- Creating webhooks in the external CRM system
- Storing webhook IDs in the webhooks table with a link back to the connectivity record

All of these responsibilities are bundled into a single function. If any step fails, subsequent steps do not execute.

### Bootstrap Function Consolidation Issue

Similarly, bootstrap operations are consolidated into one function that is responsible for:
- Creating the folder/keyspace
- Creating attributes
- Creating events

These should be separate calls so that failures in one area don't prevent progress in others.

---

## The Uninstallation Problem

### Why Uninstallation Reveals the Flaw

Uninstallation follows essentially the same flow as installation but in reverse. The critical issue arises when external systems become unreachable or credentials expire:

[Erik Andersson]:

> "On uninstallations it becomes a bit more problematic. Because on uninstallations we would want to be able to remove everything inside of Apsis regardless sometimes if the CRM system returned an external error like the API keys that we have stored might already have expired or been rotated, so we cannot actually make any requests to the CRM system."

The current approach to uninstallation:
1. Attempt to delete the API key and URL from the database
2. Attempt to delete webhooks by making a request to the CRM to delete the webhook ID
3. Only delete the webhook entry from the database *after* successful deletion in the CRM

### The Webhook Orphaning Risk

[Erik Andersson]:

> "If we were to delete it from our database [first], then there will most likely be lingering or trailing webhooks if we were to fail to delete them. If we also remove them from our database for future attempts, we would not really know what webhooks should we remove for this installation."

The reason webhooks are deleted from the CRM *before* the database is to preserve knowledge of which webhooks need cleanup. However, if the CRM API is unreachable (due to expired credentials or rotation), the current code flow will fail entirely and neither the CRM nor the database will be cleaned up.

### Cascading Failure Example

[Erik Andersson]:

> "Now then say that of course if the API key has been rotated then we will never be able to remove the webhooks from the CRM. But we would still want to remove them from within Apsis. But the way the flow is built now is that like we have one function here for like install connector or uninstall connector and that function have like all of these responsibilities and if anything any step above it here fails like the following things won't happen."

Example failure scenario:
- Uninstall begins
- Attempt to get CRM functionality fails (API key expired)
- Because the failure occurs inside a nested function, the webhook deletion steps and database cleanup steps never execute
- Webhooks remain in the Apsis database with no way to know if they were deleted from the CRM

---

## The `ignore_external_errors` Flag Problem

There is currently an `ignore_external_errors` flag that can suppress failures, but the nested function architecture makes passing this flag impractical:

[Erik Andersson]:

> "And this is of course a bit problematic because we do have a functionality today where we say like ignore external errors and if that flag is set then the function is to proceed even if an error occurred. But because these functions are so nested, like you have the install connector which calls one function, which calls one function which calls one function, we would need to pass this flag on so deep and so many like it's so many nested requests that it becomes unmanageable after a while."

---

## Proposed Solution: Flattening the Call Stack

### Core Principle

Move orchestration logic from nested helper functions into the main installation/uninstallation flow. This allows explicit error handling at each step without deep flag passing.

[Erik Andersson]:

> "What this flow aims to do is to lift like remove this nesting and make and make the installation installation flow be like the orchestrator of all of these things? So like in the installation flow you should make a explicit call to say I want to create the webhooks for this CRM system. You should not call like install connector and inside install connector you make the request. You do it in the actual installation flow because then you can handle all the errors in that main flow. You can choose to ignore those errors without having to pass any flags like 5 steps deeper in the flow."

### Refactored Structure

Instead of:
```
install_connector()
  ├─ store_credentials()
  │   └─ validate_and_store_api_key_and_url()
  └─ create_webhooks()
      └─ [nested calls]
```

The new structure should be:
```
installation_flow()
  ├─ store_credentials() [explicit call]
  ├─ [handle error, decide to proceed or abort]
  ├─ create_webhooks() [explicit call]
  ├─ [handle error, decide to proceed or abort]
  ├─ bootstrap_attributes() [explicit call]
  ├─ [handle error]
  └─ bootstrap_events() [explicit call]
```

### Bootstrap Function Splitting

[Erik Andersson]:

> "And in the same fashion here during the bootstrap, we have one function which is responsible for creating the folder, creating attributes, creating events there should be one call to bootstrap attributes and then there should be one call to bootstrap events and et cetera, et cetera. And then all those errors are handled there instead."

Currently: `bootstrap_all()` → folder + attributes + events (monolithic)

Proposed: Separate calls to `bootstrap_attributes()`, `bootstrap_events()`, etc., each with independent error handling.

### Uninstallation Benefits

With flattened calls, uninstallation becomes:

```
uninstall_flow()
  ├─ attempt_delete_webhooks_from_crm() [may fail if API key expired]
  ├─ [catch error, but continue]
  ├─ delete_webhooks_from_database()
  ├─ [handle error]
  ├─ delete_credentials_from_database()
  └─ [handle error]
```

If the CRM deletion fails, the code can still proceed to clean up the database without needing a flag passed through five levels of nesting.

---

## Scope and Impact Assessment

### Affected Connectors

[Erik Andersson]:

> "The full the installation flow. The installation flow affects everyone because all of all of the connectors have like store API credentials like all of them are split into at least two steps where like we create the webhooks in the second step and things can go wrong in the first one."

All connectors (generic, legacy, third-party) are affected by this refactoring since all require credential storage and webhook setup.

### Full Sync Producer (Not Prioritized)

The full sync producer is less critical to refactor:

[Erik Andersson]:

> "The full sync, however, is not as affected because the only real difference between the generic connector and the more legacy connectors like Dynamics is like which endpoint in the CRM system are we calling when we retrieve the data... In the case of dynamics there is one extra step and that is this mutate data to match Justin formats... But that's kind of what we are doing already."

The full sync flow is already relatively clear and stable. A separate refactoring for the full sync producer may be needed but is not as urgent.

---

## Incident Prevention vs. Maintainability

[Erik Andersson]:

> "I would not really agree, maybe as it is described, that this solution will prevent incidents that would be a bit taking it a bit too far maybe. However, it will by far simplify error handling for all of this because you can catch the errors and decide what to do with the error when you call like the create webhooks like do you want to proceed and delete things in the database or not?"

This refactoring is not a silver bullet for preventing uninstallation failures. Rather, it significantly improves the ability to handle errors gracefully when they do occur. The real benefit is maintainability and debuggability.

---

## Known Issues Motivating This Work

[Erik Andersson]:

> "This is not definitely not as urgent as the installation flow because the installation flow we already now know that there are problems on uninstallation due to like expired API credentials..."

The team has already observed real failures during uninstallation when API credentials expire or are rotated. This refactoring directly addresses that known pain point.

---

## Planned Work Items

Based on the discussion, the following stories are expected:

1. **Bootstrap refactoring**: Split the monolithic bootstrap function into separate calls for attributes, events, etc.
2. **Install connector refactoring**: Split into:
   - `store_credentials()` call
   - `create_webhooks()` call
   - Possibly additional calls for checking supported features
3. **Additional granulation**: Further analysis may reveal other consolidation opportunities (e.g., separate calls for different setup phases)

[Michal Rosikiewicz]:

> "So I see that there will be at least two stories, one for Bootstrap, one for this initial connector and two will split them in or multiply them by some things to make request to check supported features and create database entry and so on."

### Next Steps

- Erik will create an epic documenting the full scope of changes
- Erik will create technical stories for each refactoring work item
- Erik and Michal will schedule a grooming session (Friday 11:00–12:00 if schedule permits) to review code locations and finalize story definitions
- Michal requested the opportunity to read stories before the grooming meeting to familiarize himself

---

## Comparison with Other Services

[Erik Andersson]:

> "I don't know how you handle pagerduty errors in your services, but like in MA, I know we trigger pagerduty alarms if we see like a level 50 in Cloudwatch in integration we trigger it if the log level is error."

Integration currently triggers PagerDuty alerts on error-level log entries, whereas MA uses log level 50 (equivalent). This may affect how urgency is communicated when integration issues occur.

---

## Key Takeaways

1. **The Problem**: The current `install_connector` and `uninstall_connector` functions bundle credential storage, webhook creation, and bootstrap operations into monolithic functions. When any step fails, dependent steps don't execute, and error handling becomes difficult.

2. **The Pain Point**: During uninstallation, if external API credentials expire, the entire flow fails and neither the CRM nor the Apsis database gets cleaned up. The current code cannot gracefully fall back to database cleanup when the external system is unreachable.

3. **The Solution**: Flatten the call stack so the main installation/uninstallation orchestrator explicitly calls granular functions (`store_credentials()`, `create_webhooks()`, `bootstrap_attributes()`, etc.). This moves error handling to the orchestrator level where it's manageable without deep flag passing.

4. **Impact**: All connectors are affected. The refactoring improves maintainability and debuggability significantly but is not guaranteed to prevent all incidents.

5. **Priority**: Installation flow refactoring is urgent (known failures with expired credentials). Full sync producer refactoring is lower priority and requires further analysis.

6. **Scope**: At minimum, two major stories (bootstrap split, install connector split) with possible sub-tasks for checking supported features and other steps.

---

## Unresolved Questions

- [Erik]: "The second one, I have actually no idea what this is about. I'm gonna need to check what this might actually be..." — Erik needs to analyze a second diagram/story from Benjamin to understand its requirements. He suspects the functionality may already exist and needs clarification.
- The full scope of the full sync producer refactoring is unclear and deferred for later analysis.
- Specific file paths and class names for the refactoring will be identified during the Friday grooming session.

## Action Items

- **Erik**: Create an epic and problem statements for the refactoring work by end of week
- **Erik**: Create technical stories for bootstrap refactoring and install connector refactoring
- **Erik**: Analyze the second story/diagram from Benjamin to clarify intent
- **Erik & Michal**: Schedule grooming session (proposed Friday 11:00–12:00) to review code and finalize stories
- **Michal**: Review stories before grooming meeting to familiarize himself