---
source_file: Integration installation story grooming.txt
domain: Apsis One Integrations
topics: [integration installation flow, uninstall flow refactoring, webhook registration/unregistration, connector interface design, error handling flags, bootstrapping flow, story grooming and estimation]
speakers: [Erik Andersson (domain expert/architect), Tomasz Kowalski (developer), Michal Rosikiewicz (developer), Lukasz Grabowski (team lead)]
key_components: [Integration Manager, FSC Enterprise Connector, Dynamics Connector, Lime Connector, Generic Connector, Connector Installer Interface, Operations Bus, Webhook Registration, Bootstrap Flow]
session_type: knowledge-transfer
---

## Session Overview

This session is a story grooming and knowledge transfer for a planned refactoring of the integration installation/uninstallation flow in Apsis One. Erik Andersson walks the team through the core problem: the current installation and uninstallation logic is deeply nested, with too many responsibilities in single functions, making error handling cumbersome — particularly around webhook unregistration when external CRM credentials have already been rotated. The proposed solution is to flatten the call hierarchy by moving discrete operations (register webhooks, unregister webhooks, store credentials, remove credentials, bootstrap folder/attributes/events) into the Integration Manager layer, where errors can be handled directly. The session results in several groomed stories under a refactoring epic, with rough estimates assigned.

---

## Current Problem: Deeply Nested Installation/Uninstallation Functions

### Overview of the Problem

The core issue is that the current install and uninstall functions have **too many responsibilities** and are nested four to five levels deep. Arguments — including error-handling flags — must be passed down through every level of the call stack, making the code brittle and difficult to maintain.

**Erik Andersson:**
> A lot of the functions have way too much responsibilities. They are nested four to five steps. You need to pass on the arguments a lot, and above all, the error handling becomes very clunky.

### The Uninstall Flow Today (FSC Enterprise Example)

The current call chain for uninstalling FSC Enterprise looks like:

1. User clicks "Uninstall FSC Enterprise"
2. **Integration Manager** calls `uninstall` (from the connector `installer` interface)
3. The FSC Enterprise uninstall function calls `unregisterWebhooks()` — an external call to the CRM
4. Inside `unregisterWebhooks()`, webhook subscription IDs and related data are also **deleted from the local database**
5. After `unregisterWebhooks()` returns, the FSC Enterprise uninstall function makes a SQL call to delete the **credential entries** from the database

**The critical failure scenario:** If the customer has already rotated their API key in the CRM (e.g., to prepare for removing the integration), the call to `unregisterWebhooks()` will receive a `401` or `403` from the CRM. The function throws an error and **halts execution**. The database cleanup of the credential entries is never reached, leaving orphaned data in the database.

### The `ignore_external_system_errors` Flag Problem

To handle scenarios where CRM errors should not abort the flow (e.g., account deletion via the **Operations Bus**), a flag — `ignore_external_system_errors` — was introduced. The intent is: even if we fail to unregister webhooks in the CRM, we should still clean up our own database.

**The problem:** This flag must be threaded through every level of the nested call stack:

- Integration Manager sets the flag on the `uninstall` call
- `uninstall` passes it into the connector's uninstall function
- The connector uninstall function passes it into `unregisterWebhooks()`
- `unregisterWebhooks()` passes it into the function that deletes the webhook DB entries

> "You have the uninstall function in the Integration Manager, which calls an uninstall function, which calls an unregister webhooks function, which calls something which calls something more."

### Two Distinct Scenarios That Trigger `ignore_external_system_errors`

1. **Operations Bus account delete:** When an account is being deleted, we want to do a best-effort cleanup — remove all our data even if CRM webhook unregistration fails. Leaving this incomplete would cause the system to continuously send failure requests, which was agreed to be unacceptable.

2. **Customer-initiated force-uninstall:** If a customer has accidentally rotated their API key and cannot uninstall through the normal flow, a **direct request to the Integration Manager** can be made to override the default behaviour and force-remove the integration. Erik mentioned that the team handles these requests occasionally and will share the relevant **Postman collection** (collection name: `integration flow`).

**Important default behaviour:** By default, the uninstallation process **aborts entirely** on any error. This is intentional — leaving trailing webhooks active after a failed uninstall could result in the CRM continuing to send private customer data to Apsis, which is a **security concern**.

---

## Connector Interface Architecture

### The `installer` Interface

Each connector implements a set of interfaces defined in:

```
libraries/connectors/types
```

This is where the contract for each connector is defined — what functions each connector **must** implement. If a connector does not support a specific function, it can return `nil`.

The current interface includes:
- `install` — called during installation
- `uninstall` — called during uninstallation

### Current Bootstrap Flow

Currently, the Integration Manager calls a single `bootstrap` function, which internally handles:
- Creating the custom folder (if it doesn't exist)
- Bootstrapping all custom attributes (if they don't exist)
- Bootstrapping all events (if they don't exist)

This is another example of a single function with too many responsibilities, making granular error handling difficult.

---

## Proposed Refactoring: Flatten the Call Hierarchy

### Core Architectural Change

The proposed solution is to **move discrete operations up into the Integration Manager** so that it directly calls small, focused functions on each connector, rather than delegating to one large install/uninstall function per connector.

Instead of:
```
IntegrationManager.uninstall()
  └── EnterpriseConnector.uninstall()
        ├── unregisterWebhooks()   ← also deletes DB entries
        └── deleteCredentials()
```

The Integration Manager would directly call:
```
IntegrationManager.uninstall()
  ├── connector.unregisterWebhooks()
  ├── connector.removeCredentials()
  └── [handle errors directly at this level]
```

Similarly for installation:
```
IntegrationManager.install()
  ├── connector.createFolder()
  ├── connector.bootstrapAttributes()
  ├── connector.bootstrapEvents()
  ├── connector.storeCredentials()
  └── connector.registerWebhooks()
```

**Key benefit for error handling:** The `ignore_external_system_errors` flag no longer needs to be threaded down through the stack. It is handled directly in the Integration Manager at the call site of each discrete function.

**Additional benefit noted by Michal:** These discrete operations could potentially be run **in parallel**, which would speed up the installation/uninstallation flow.

### New Functions to Add to the Connector Interface

The following new functions are proposed for the connector installer interface:

| Function | Notes |
|---|---|
| `registerWebhooks()` | Called from Integration Manager during install |
| `unregisterWebhooks()` | Called from Integration Manager during uninstall; must handle `ignore_external_system_errors` |
| `storeCredentials()` | Stores API credentials to DB |
| `removeCredentials()` | Removes API credentials from DB |
| `createFolder()` | Extracted from `bootstrap` |
| `bootstrapAttributes()` | Extracted from `bootstrap` |
| `bootstrapEvents()` | Extracted from `bootstrap` |

### Critical Implementation Warning: Interface Changes Affect All Connectors

> "As soon as you change the installer interface, you are adding a mandatory function to all of these connectors."

Adding `registerWebhooks()` to the interface means it must be implemented on **every connector** that currently registers webhooks:
- Dynamics Connector
- FSC Enterprise Connector
- Lime Connector
- Generic Connector
- (and others — "Intermale" mentioned)

**Erik's suggested approach:** Start with FSC Enterprise as it is a simpler connector and a good learning exercise. Once that pattern is established, apply the same refactoring to the other connectors. The work can be split across developers — e.g., one developer handles installation-side changes, another handles deletion/uninstallation-side.

---

## Error Handling Behaviour by Context

| Context | Behaviour |
|---|---|
| User-initiated uninstall (normal flow) | **Abort on any error** — do not remove DB entries if CRM cleanup fails (security default) |
| Operations Bus account delete | **Ignore external errors** — best-effort cleanup, remove all DB entries even if CRM calls fail |
| Force-uninstall (customer support case) | **Ignore external errors** — override via direct Integration Manager request (Postman collection: `integration flow`) |

**Note on installation flow:** The `ignore_external_system_errors` flag is only relevant to the **uninstall** flow. During installation, failures should always abort — there is no equivalent ambiguity.

---

## Stories Created and Estimates

The session resulted in the creation of the following stories under a refactoring epic. Estimates are rough and were openly acknowledged as uncertain — the team agreed to set initial estimates and adjust as they gain familiarity with the code.

| Story | Estimated Points | Notes |
|---|---|---|
| Add `registerWebhooks()` to connector installer interface + implement in all connectors | 5 | Work can be parallelised across connectors once interface is defined |
| Add `unregisterWebhooks()` to connector installer interface + implement in all connectors | 5 | Most critical — this is where the `ignore_external_system_errors` flag is needed |
| Add `storeCredentials()` and `removeCredentials()` to interface | ~5 | |
| Refactor `bootstrap` into `createFolder()` / `bootstrapAttributes()` / `bootstrapEvents()` | 2 | Lower priority; does not involve connector-specific logic; not the source of production issues |

**Estimation discussion:** Erik acknowledged that for him the work might take 1-2 hours per connector, but for the team it would likely be 2–3 days per connector given unfamiliarity. The estimate of 5 points per story is an aggregate. The team agreed to not stress about going over estimate on these stories, as the primary purpose is KT and code familiarity.

---

## Key Takeaways

1. **The root problem** is a "God function" anti-pattern in each connector's `install`/`uninstall` methods — they handle CRM calls, DB operations, and error logic all in one place.
2. **The fix** is to flatten the hierarchy: the Integration Manager should orchestrate discrete steps, each in its own interface function, and handle errors at that level.
3. **The `ignore_external_system_errors` flag** is the most concrete pain point — it currently has to be threaded through 4+ levels of function calls and is the direct cause of production issues with orphaned DB entries after failed uninstalls.
4. **Security implication of the default:** Failing to fully uninstall (leaving trailing webhooks) can result in a CRM continuing to push private customer data to Apsis — this is the reason the default is to abort on error rather than proceed.
5. **Interface changes have wide blast radius** — adding any new function to the connector installer interface immediately requires implementation across all connectors (Dynamics, FSC Enterprise, Lime, Generic, etc.).
6. **FSC Enterprise** is the recommended starting point for implementation — simpler than other connectors and a good learning exercise.
7. **Bootstrapping refactoring** (splitting `bootstrap` into `createFolder`/`bootstrapAttributes`/`bootstrapEvents`) is lower priority — it has not caused production incidents, unlike the webhook unregistration flow.

---

## Unresolved Questions and Action Items

- [ ] **Erik** to polish the created stories and add descriptions with subtask breakdowns in story descriptions.
- [ ] **Erik** to add additional technical refinement stories to the epic.
- [ ] **Erik** to share the **Postman collection** (`integration flow`) containing the direct Integration Manager requests for force-uninstall scenarios.
- [ ] Team to decide on connector assignment for parallel implementation (suggested split: one developer on install-side, one on uninstall-side).
- [ ] **⚠️ Ambiguity:** The transcript references connectors including "Intermale" — this may be a transcription artifact for another connector name. Verify the full list of connectors that implement the installer interface in `libraries/connectors/types`.
- [ ] Prioritisation of this epic relative to incoming product work is not yet decided — dependent on product priorities communicated to the team.