---
source_file: "Erik - Refactor Justin connectivity flows.txt"
domain: Apsis One Integrations
topics: [installation flow refactoring, uninstallation error handling, connector installation steps, bootstrap process, webhook management, full sync producer, error handling patterns, story grooming]
speakers: ["Erik Andersson (Senior/Outgoing Developer)", "Michal Rosikiewicz (Incoming Developer)"]
key_components: [installation flow, uninstall connector, install connector, bootstrap, webhook management, FSC Enterprise connector, Microsoft Dynamics connector, generic connector, full sync producer, Apsis One API]
session_type: knowledge-transfer
---

## Session Overview

Erik walks Michal through a planned refactoring of the integration installation/uninstallation flow in Apsis One Integrations. The core problem is that key functions in the installation flow have too many responsibilities and are deeply nested, making error handling — particularly during uninstallation with expired credentials — fragile and difficult to manage. The session covers the current flow's structure, the specific failure modes that motivate the refactor, and a proposed decomposition into more granular, independently callable steps. They also briefly touch on the full sync producer refactor story, which Erik notes he needs to re-examine before discussing further.

---

## Installation Flow Overview: What Happens During Installation

The installation flow is identified as the most step-heavy flow across all integration services — not necessarily the most complex to understand, but the one with the most discrete steps involved.

### Steps in the Current Installation Flow

When a connector is installed (e.g., Microsoft Dynamics, FSC Enterprise), the following occurs:

1. **Key space creation**: A dedicated key space is created for the CRM. Examples:
   - Installing Microsoft Dynamics → creates a Dynamics key space
   - Installing FSC Enterprise → creates an FSC Enterprise key space

2. **Bootstrap**: Some CRM systems have a predefined schema specifying which attributes and events should exist in Apsis upon installation. If the integration includes such a schema, these attributes and events are bootstrapped automatically at install time.

3. **Default mapping setup**: Default field mappings are configured — for example, mapping the CRM's contact ID field to the corresponding Apsis field (e.g., `Contact ID` for Microsoft Dynamics).

4. **Resource generation**: An Apsis One API key is generated and sent to the CRM system, along with:
   - Section discriminator
   - Section ID
   - Key space discriminator and ID

   This allows the CRM to use the A1 API in any custom flows it may have.

5. **Connectivity registration**: CRM connection details (API URL, credentials) are stored in the database. The specifics depend on which CRM is being installed.

---

## The Core Problem: Over-Nested Functions and Insufficient Granularity

### The `install_connector` / `uninstall_connector` Problem

The **`install_connector`** function (and its counterpart `uninstall_connector`) is the biggest culprit. Using FSC Enterprise as an example:

- The customer enters an API URL and API key in the UI.
- These are stored in the database (in a table referred to as the `con_fsc` table — connections for FSC — storing the API URL and API key).
- Within the **same function**, the code proceeds to create webhooks in the CRM system by making an external request, receiving a webhook ID back, and storing that ID in a webhooks table linked to the connectivity record just created.

The problem: **all of these responsibilities live in one monolithic function**, and they call sub-functions which call further sub-functions. If any step fails, everything after it in that chain does not execute.

### Why This Is Especially Problematic During Uninstallation

During uninstallation, the system attempts to:
1. Delete the stored API key and URL from the database.
2. Delete webhooks — but only **after** successfully requesting the CRM to delete the corresponding webhook on its side.

The rationale for step 2's ordering:

> "If we were to delete it from our database [first], then there will most likely be lingering or trailing webhooks. If we also remove them from our database for future attempts, we would not really know what webhooks should we remove for this installation."

**The failure scenario**: API keys stored at install time may have expired or been rotated by the time uninstallation is attempted. This means the system **cannot make requests to the CRM** to delete webhooks. But the current flow design means that if the external CRM call fails, the database records for those webhooks are never cleaned up either — even though we'd want to remove them from Apsis regardless of whether the CRM-side deletion succeeded.

### The `ignore_external_errors` Flag: A Workaround That Doesn't Scale

There is an existing flag (`ignore_external_errors`) that, when set, tells a function to proceed even if an error occurred. However, because the functions are so deeply nested:

> "We would need to pass this flag so deep and so many nested requests that it becomes unmanageable after a while."

---

## Proposed Refactor: Lift Responsibilities Into the Main Flow

### Core Principle

The installation flow should become the **orchestrator** of all steps. Instead of calling `install_connector` and having that function internally make CRM requests and database writes, the main installation flow should make **explicit, separate calls** for each discrete action.

**Example — current (problematic) pattern:**
```
installation_flow()
  └─ install_connector()
       ├─ store_credentials_in_db()
       └─ create_webhooks_in_crm()  ← nested, hard to handle errors separately
```

**Example — proposed pattern:**
```
installation_flow()
  ├─ store_credentials_in_db()     ← explicit call, error handled here
  └─ create_webhooks_in_crm()      ← explicit call, error handled here independently
```

This way, errors at any step can be caught and acted on at the top level — for instance, deciding whether to proceed with database cleanup even if a CRM webhook deletion fails — **without needing to propagate flags deep into nested calls**.

### Bootstrap Function Decomposition

The same principle applies to the bootstrap section. Currently there is a single "God function" responsible for creating the folder, attributes, and events. The proposed change:

- One explicit call to **bootstrap attributes**
- One explicit call to **bootstrap events**
- (etc.)

Each call's errors are handled independently in the main flow.

### Clarification on Scope of Impact

[Erik Andersson]: This refactor affects **all connectors**, not just one. Every connector has at least a "store API credentials" step and a "create webhooks" step — and things can go wrong in the first step that currently prevent the second from being reached (or cleaned up).

> "I would not really agree, maybe as it is described, that this solution will prevent incidents — that would be taking it a bit too far maybe. However, it will by far simplify error handling for all of this."

---

## Story Breakdown: What Gets Created

Based on the discussion, at minimum two stories are identified:

1. **Bootstrap decomposition story**: Split the single bootstrap God function into separate, explicit calls (e.g., bootstrap attributes, bootstrap events).

2. **Install connector decomposition story**: Split `install_connector` (and `uninstall_connector`) into at minimum:
   - `store_credentials` call
   - `create_webhooks` call

[Erik Andersson]: Plans to review the full installation flow again before the grooming session to identify any additional decomposition opportunities beyond these two. These two are confirmed as the biggest known culprits, with `install_connector` being the one that has already caused known production problems.

---

## Full Sync Producer Refactor Story: Status Unclear

A second story in the epic relates to refactoring the **full sync producer**. Erik was candid that he does not fully understand what the story intends, because in his view, the behavior described is already implemented.

### Current Understanding of Full Sync: Generic vs. Legacy Connectors

The main difference between the **generic connector** and legacy connectors like **Microsoft Dynamics** in the full sync flow:

- **Generic connector**: Data is retrieved directly from the CRM endpoint in the format Apsis requires (one step).
- **Microsoft Dynamics**: Two steps:
  1. Retrieve data from the CRM (`get data` step)
  2. Transform/mutate data to match the required format (e.g., Dynamics returns "apples and pears" but the system needs a normalized format — Erik used the analogy: *"Dynamics gives us apples and pears and instead we want a Ferrari back when we process it"*)

[Erik Andersson]: This two-step pattern is already what the code does. He suspects there is something in the story diagram he is misreading or missing context on.

### Priority Assessment

> "This is definitely not as urgent as the installation flow, because the installation flow we already know there are problems on uninstallation due to expired API credentials, whereas the full sync producer is rather stable and quite clear. If there were to be any errors, you will easily see where those problems happen."

---

## Logging and PagerDuty Error Levels (Brief Aside)

A brief discussion on alerting behavior across services:

- In **MA (Marketing Automation)**: PagerDuty alarms are triggered by log level 50 in CloudWatch.
- In **Integration**: PagerDuty alarms are triggered if the log level is `ERROR`.
- Michal's service uses log levels defined in code (errors and warnings).

[Erik Andersson] noted there is a potential story around changing specific `error`-level log entries to `warning` where appropriate, but these are typically handled ad hoc as they arise rather than requiring dedicated grooming.

---

## Process: Story Grooming Plan

- Erik will create the **epic** with the problem description and the technical stories inside it.
- Michal requested the ability to read stories before the grooming meeting to familiarize himself with the context.
- A 1–1.5 hour grooming session was discussed, during which Erik will show the relevant code locations directly.
- The full sync producer story will **not** be covered in the next session — Erik needs to re-analyze the diagram first.

---

## Key Takeaways

1. **The installation flow is the most step-heavy flow in the integrations domain** and is the highest-priority target for refactoring.
2. **`install_connector` / `uninstall_connector` are God functions** — they own too many responsibilities (store credentials, create webhooks, etc.) in a deeply nested call chain. This is a known source of production problems, specifically during uninstallation when CRM API credentials have expired.
3. **The proposed fix is architectural**: promote all discrete steps to explicit top-level calls in the installation/uninstallation orchestrator, so error handling decisions can be made at the top level without propagating flags through layers of nesting.
4. **The bootstrap function has the same problem** and should be decomposed into separate calls per resource type (attributes, events, etc.).
5. **The `ignore_external_errors` flag exists** but is not a viable long-term solution given the current nesting depth.
6. **The full sync producer story is lower priority** and its intent needs to be clarified by Erik before it can be groomed.
7. **All connectors are affected** by the installation flow refactor — this is not scoped to a single CRM integration.

---

## Unresolved Questions / Action Items

- [ ] **Erik**: Review full installation flow code to identify any additional decomposition opportunities beyond bootstrap and `install_connector`.
- [ ] **Erik**: Create the epic, problem description, and draft technical stories.
- [ ] **Erik**: Re-examine the full sync producer story/diagram to understand what the original author intended — it may already be implemented or there may be a nuance being missed.
- [ ] **Erik + Michal**: Schedule ~1–1.5 hour grooming session (tentatively Friday 11:00–12:00) where Erik will walk through the code locations in depth.
- [ ] **Michal**: Review stories before the grooming meeting once Erik creates them.
- ⚠️ **Ambiguity**: The `con_fsc` table name was stated with uncertainty by Erik ("I think called con FSC table"). The exact table name should be confirmed in the codebase.