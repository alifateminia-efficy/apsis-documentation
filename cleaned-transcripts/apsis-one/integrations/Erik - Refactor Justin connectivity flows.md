---
source_file: Erik - Refactor Justin connectivity flows.txt
domain: Apsis One Integrations
topics: [Installation flow refactoring, Connector installation steps, Webhook management, Bootstrap process, Uninstallation error handling, Credentials storage, Full sync producer]
speakers: [Erik Andersson (senior/outgoing developer), Michal Rosikiewicz (incoming developer)]
key_components: [Installation flow, Generic connector, Legacy connector (Microsoft Dynamics), FSC Enterprise connector, Webhook table, Bootstrap schema, Keyspace, A1 API key, Full sync producer]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors"]
---

## Session Overview

Erik walks Michal through a planned refactoring of the connector installation flow in Apsis One Integrations. The core problem is that the current installation (and uninstallation) flow has deeply nested functions with too many responsibilities, making error handling difficult — particularly during uninstallation when external API credentials may have expired. Erik proposes lifting these nested steps into explicit, flat calls in the main installation flow so that errors can be caught and acted upon at the top level. The session also briefly touches on the full sync producer refactoring story, which Erik has not fully analyzed yet.

---

## Overview of the Current Installation Flow

The installation flow is described as the most step-heavy flow across all integrations services. When a connector is installed, the following steps occur:

1. **Keyspace creation** — A CRM-specific keyspace is created (e.g., a Dynamics keyspace for Microsoft Dynamics, an FSC Enterprise keyspace for FSC Enterprise).
2. **Bootstrap** — Some CRM systems define a predefined schema of attributes and events that should exist in Apsis. If the integration has such a schema, those attributes and events are created at install time.
3. **Default mapping setup** — For example, the CRM ID in Apsis is mapped to the configured ID field on contacts or persons (e.g., `Contact ID` for Microsoft Dynamics).
4. **Resource generation** — An A1 API key is generated and sent to the CRM system, along with section discriminator, section ID, keyspace discriminator, and keyspace ID, so the CRM can use the Apsis One API in its custom flows.
5. **Connectivity registration** — The API URL and credentials for the CRM system are stored in the database. The specific storage varies by CRM.

---

## The Core Problem: Overly Nested Functions with Excessive Responsibility

### Install Connector Function

The biggest problem is the `install connector` function (and its counterpart `uninstall connector`). Using FSC Enterprise as an example, the current behavior is:

1. The customer enters an API URL and API key in the UI.
2. The `install connector` function stores these in the connections table (e.g., `con_fsc` or similar — Erik's exact table name was uncertain).
3. Within the same function, a request is made to the CRM to generate webhooks.
4. The returned webhook ID is stored in a webhooks table, linked to the connectivity entry created in step 2.

> [Erik]: "Because these functions are so nested — you have install connector which calls one function, which calls one function which calls one function — we would need to pass this flag on so deep and so many nested requests that it becomes unmanageable after a while."

### The `ignore_external_errors` Flag Problem

There is an existing `ignore_external_errors` flag that, when set, tells a function to proceed even if an error occurred. However, because of the deep nesting, this flag must be threaded through many layers of calls, which is unmanageable.

---

## Why Uninstallation Makes This Worse

During uninstallation, the system tries to:
- Delete the stored API URL and API key from the database.
- Delete webhooks — but **only after** successfully requesting the CRM system to delete the corresponding webhook by ID.

**The rationale for the two-step webhook deletion:** If the database entry is deleted before the external webhook is removed and the external call fails, there will be no record of which webhooks to retry deleting in future attempts. This is intentional design.

**The problem this creates:** If the customer's API key has been rotated or expired, no requests can be made to the CRM system. This means the webhook cannot be deleted externally, and therefore the webhook record is never cleaned up from the Apsis database either — even though it should be, since we cannot do anything with it.

> [Erik]: "We would still want to remove them from within Apsis [even if the external delete fails]. But the way the flow is built now is that the function has all of these responsibilities and if any step above fails, the following things won't happen."

This is a known, real problem: uninstallations fail to fully clean up when API credentials have expired.

---

## Proposed Refactoring: Flatten the Installation Flow

### Principle

The installation flow should become the **orchestrator** of all steps. Instead of `install connector` internally calling sub-functions that call further sub-functions, the main installation flow should make **explicit, flat calls** to each step:

- Call: store credentials
- Call: create webhooks
- Call: bootstrap attributes
- Call: bootstrap events
- etc.

This way, errors from each call are handled directly in the main flow without needing to pass flags deep into nested calls. The caller can decide: "if create webhooks fails, do I proceed with the rest or not?"

### Specific Refactoring Targets Identified

1. **Bootstrap function** — Currently one "god function" creates the folder, attributes, and events. Should be split into:
   - One call to bootstrap attributes
   - One call to bootstrap events
   - (Possibly one call per logical unit)

2. **Install connector function** — Currently handles both credential storage and webhook creation. Should be split into:
   - One call: store credentials (API URL + API key → database)
   - One call: create webhooks (request to CRM + store returned webhook ID)

> [Erik]: "This will by far simplify error handling for all of this, because you can catch the errors and decide what to do with the error when you call 'create webhooks' — do you want to proceed and delete things in the database or not?"

Erik clarifies the scope of impact: this affects **all connectors**, not just generic ones, because all connectors have at least the credential storage + webhook creation split.

---

## Story Grooming Plan

- Erik will create the epic and problem descriptions.
- Stories will be groomed together in a dedicated session (estimated 1–1.5 hours).
- Erik will show Michal the relevant code locations during that session.
- At minimum, two stories are anticipated:
  1. Refactor bootstrap function into separate calls
  2. Refactor install connector into separate `store credentials` + `create webhooks` calls
- Erik intends to review the full installation flow again to identify any additional refactoring opportunities.

---

## Full Sync Producer Refactoring (Second Story — Unresolved)

There is a second story in the epic related to the full sync producer. Erik has not fully analyzed it and is uncertain what additional refactoring is intended, because in his view the current implementation already does what the story describes.

What is known about the current full sync behavior:
- **Generic connector:** Data is retrieved directly from the CRM in the required format (the generic connector spec mandates the format), so it is one step.
- **Legacy connector (e.g., Microsoft Dynamics):** Data retrieval is two steps: (1) fetch data from Dynamics, (2) transform/mutate the data to match the expected internal format.

> [Erik]: "In the case of Dynamics it is two steps — first we do the get data step, then we transform it. But that's kind of what we are doing already. So I don't... there must be something I am missing that he intended with this diagram."

Erik notes the full sync producer is **more stable and lower priority** than the installation flow refactoring.

---

## Key Takeaways

- The installation flow is the most complex flow in integrations and currently has tightly coupled, deeply nested functions with too many responsibilities per function.
- The immediate real-world consequence is that uninstallations fail to clean up Apsis-side data when CRM API credentials have expired, because the nested structure prevents partial success.
- The fix is to flatten the flow so the main installation/uninstallation orchestrator makes explicit calls to each discrete step, enabling per-step error handling.
- All connector types are affected by the install connector refactoring (not just generic connectors).
- The full sync producer story is lower priority and needs further analysis by Erik before it can be groomed.

---

## Unresolved Questions / Action Items

- **Erik:** Review the full installation flow code to identify any additional refactoring candidates beyond the two main stories.
- **Erik:** Create the epic and problem descriptions in the backlog; list the stories before the grooming meeting.
- **Erik + Michal:** Schedule a ~1–1.5 hour grooming session (tentatively Friday 11:00–12:00) where Erik will walk through the code locations.
- **Erik:** Analyze the full sync producer story diagram more carefully — it is currently unclear what additional refactoring is intended beyond what is already implemented.