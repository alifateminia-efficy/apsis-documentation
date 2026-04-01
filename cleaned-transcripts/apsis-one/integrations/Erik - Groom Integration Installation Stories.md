---
source_file: Erik - Groom Integration Installation Stories.txt
domain: Apsis One Integrations
topics: [installation flow refactoring, uninstall flow, webhook registration/unregistration, error handling, connector interfaces, ignore-external-system-errors flag, bootstrapping, operations bus, story grooming]
speakers: ["Erik Andersson (Senior/Lead Developer, domain expert)", "Tomasz Kowalski (Developer)", "Michal Rosikiewicz (Developer)", "Lukasz Grabowski (Team Lead/Scrum Master)"]
key_components: [Integration Manager, FSC Enterprise Connector, Dynamics Connector, Lime Connector, Generic Connector, Installer Interface, Operations Bus, Webhook Registration/Unregistration, Bootstrap Logic]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson led a grooming/knowledge-transfer session covering a planned refactoring epic for the Groom (integration) installation and uninstallation flow. The core problem is that current connector uninstall functions have too many nested responsibilities, making error handling — particularly around the `ignore_external_system_errors` flag — extremely cumbersome to propagate through the call stack. Erik walked through the current architecture in code, explained the specific failure scenarios that motivated this refactoring, and groomed a set of stories to decompose monolithic install/uninstall functions into discrete, independently-callable steps in the Integration Manager. Story estimation was discussed and rough sizing was agreed upon.

---

## Current Architecture: The Problem with Monolithic Install/Uninstall Functions

### Overview of the Current Flow

The current installation/uninstallation flow has functions with too many responsibilities. Functions are nested four to five levels deep, requiring arguments (especially error-handling flags) to be passed down through every level of the call stack. This makes error handling clunky and brittle.

The flow today for uninstall looks roughly like:

```
Integration Manager: uninstall()
  └─ calls installer.uninstall()  [connector-specific, e.g. FSC Enterprise]
        └─ calls unregisterWebhooks()
              └─ calls deleteWebhookDataFromDB()
        └─ calls deleteCredentialsFromDB()
```

### Connector Interface Definition

Each connector is defined as a set of **interfaces** — specifying which functions must be implemented for the connector to be valid. If a connector doesn't support a specific function, it can return nil.

The interface definitions live at:

```
libraries/connectors/types
```

The current interface defines two primary functions:
- `install` — called during installation
- `uninstall` — called during uninstallation

Each connector (FSC Enterprise, Dynamics, Lime, Generic, etc.) implements these interfaces.

### Concrete Example: FSC Enterprise Uninstall Failure Scenario

In the FSC Enterprise connector's `uninstall` function:
1. It calls the CRM to unregister webhooks (external HTTP request)
2. Inside the unregister webhooks function, it also deletes webhook subscription IDs from the local database
3. After that, it makes a SQL request to delete the stored API credentials from the database

**The failure scenario:** If the customer has already rotated or deleted the API key in the CRM before triggering uninstall, the call to unregister webhooks will return a `401` or `403`. Because an error is encountered, the function halts — the credential database entries are **never deleted**, leaving orphaned data in the database.

> "The uninstall function should not both make the request to remove the webhooks and remove the webhook data and remove the credentials database entries."

---

## The `ignore_external_system_errors` Flag: Why It Exists and Why Propagating It Is Painful

### What the Flag Does

By default, the uninstallation process **aborts completely** if any error is encountered. This is intentional: if webhooks remain registered in the CRM after we've deleted our local records, the CRM could continue sending private customer data to Apsis One after the customer has technically stopped using the integration — an information security risk.

However, there are two scenarios where you want to **ignore CRM errors and proceed with local cleanup regardless**:

1. **Operations Bus account delete events:** When an account is being deleted via the operations bus, Apsis One must clean up all its own data even if CRM calls fail. If cleanup is blocked by a CRM error, the platform will continuously send failure responses — which has been agreed is unacceptable. Best-effort cleanup is the correct behavior here.

2. **Force-uninstall by support/admin:** Customers occasionally accidentally rotate API keys and then cannot reinstall because they cannot uninstall (the uninstall fails due to the invalid credentials). A direct API request to the Integration Manager can override the default behavior and force-remove the integration. Erik noted he would share the relevant Postman collection (collection name: **`integration flow`**).

### Why the Flag Is Hard to Propagate Today

When the flag is set at the Integration Manager level (e.g., triggered by an operations bus delete event), it must currently be passed:

```
Integration Manager: uninstall(ignoreExternalSystemErrors=true)
  └─ connector.uninstall(ignoreExternalSystemErrors=true)
        └─ unregisterWebhooks(ignoreExternalSystemErrors=true)
              └─ deleteWebhookDataFromDB(ignoreExternalSystemErrors=true)
```

Every nested function must accept and forward the flag. Adding a new nested function anywhere in the chain requires updating the entire chain. This is the primary maintainability problem this refactoring addresses.

> "This flag of course needs to be continuously passed down through every nested function that we have."

---

## Proposed Refactoring: Flatten the Call Stack in the Integration Manager

### Core Principle

Instead of one "god function" per connector that handles the entire install or uninstall lifecycle, the **Integration Manager itself** should call discrete, single-responsibility functions in sequence. Each step is called explicitly and its errors are handled at that level — no flag propagation needed.

**Current approach (conceptual):**
```
integrationManager.uninstall()
  └─ enterpriseConnector.uninstall()  // handles everything internally
```

**Target approach (conceptual):**
```
integrationManager.uninstall()
  ├─ enterpriseConnector.unregisterWebhooks()   // CRM call
  ├─ enterpriseConnector.removeCredentials()    // DB cleanup
  └─ [handle ignoreExternalSystemErrors here, at the top level]
```

Similarly for install:
```
integrationManager.install()
  ├─ enterpriseConnector.createFolder()
  ├─ enterpriseConnector.bootstrapAttributes()
  ├─ enterpriseConnector.bootstrapEvents()
  ├─ enterpriseConnector.registerWebhooks()     // CRM call
  └─ enterpriseConnector.storeCredentials()     // DB write
```

### Additional Benefit: Potential Parallelism

[Michal Rosikiewicz] raised that splitting into discrete steps could also allow some steps to run in parallel, which would speed up the uninstallation flow. Erik confirmed this is not implemented today but is a valid future optimization.

---

## Proposed New Connector Interface Functions

The refactoring requires adding new functions to the **connector installer interface** (`libraries/connectors/types`). The following functions are proposed:

| Function | Direction | Notes |
|---|---|---|
| `registerWebhooks` | Install | Makes CRM call to register; separated from credential storage |
| `storeCredentials` | Install | Persists API credentials to DB |
| `unregisterWebhooks` | Uninstall | Makes CRM call to deregister; **must respect ignore-external-errors flag** |
| `removeCredentials` | Uninstall | Deletes credentials from DB |
| `createFolder` | Install/Bootstrap | Creates custom folder if not exists |
| `bootstrapAttributes` | Install/Bootstrap | Bootstraps custom attributes if not exists |
| `bootstrapEvents` | Install/Bootstrap | Bootstraps custom events if not exists |

### Important Implementation Caveat: Interface Changes Affect All Connectors

> "As soon as you change the installer interface, that means you are now adding a mandatory function to all of these connectors."

Adding any function to the interface requires implementing it in **every connector**: Dynamics, FSC Enterprise, Lime, Generic Connector, and any others that currently register webhooks. The interface change itself takes minutes; implementing the logic across all connectors takes days.

**Recommended approach:** Start with **FSC Enterprise** as it is the simpler connector, implement the full pattern there first to validate the approach, then roll out to the other connectors.

**Note on `unregisterWebhooks`:** This is the most critical function to implement correctly with respect to the `ignore_external_system_errors` flag. If a CRM call fails during unregistration, the function must still proceed to clean up the local DB records when the flag is set.

**Note on bootstrap functions (`bootstrapAttributes`, `bootstrapEvents`):** These are lower priority than the webhook registration/unregistration functions. The webhook unregistration flow has caused actual production issues; the bootstrap path has not caused significant problems.

---

## Error Handling Behavior: Install vs. Uninstall

An important asymmetry:

- **During installation:** Errors should **not** be ignored. If any step fails, the installation should abort. There is no scenario where we want partial installations silently succeeding.
- **During uninstallation:** Errors from the external CRM system can optionally be ignored (via the `ignore_external_system_errors` flag), but local DB cleanup should always proceed when the flag is set.

---

## Story Breakdown and Estimation

Stories were groomed in Jira under an epic. The stories map to the proposed interface functions above. Agreed approach for estimation:

- Stories involving adding logic to **each connector individually** (e.g., `registerWebhooks`, `unregisterWebhooks`) are larger because the same logic must be implemented across all connectors.
- Stories that are **connector-agnostic** (e.g., bootstrap refactoring in the Integration Manager) are smaller.
- The team agreed to use story points as reference points rather than precise estimates.

**Rough estimates discussed:**

| Story | Estimate | Notes |
|---|---|---|
| `registerWebhooks` / `unregisterWebhooks` across all connectors | ~5 points each | Can be parallelized: one dev handles install side, one handles uninstall side |
| `storeCredentials` / `removeCredentials` | ~5 points | Similar connector-by-connector effort |
| Bootstrap refactoring (`createFolder`, `bootstrapAttributes`, `bootstrapEvents`) | ~2 points | Connector-agnostic, done in one place; `bootstrapAttributes` and `bootstrapEvents` can be combined into one story |

Erik noted he would also create additional **technical refinement stories** to be added to the epic after the session.

[Lukasz Grabowski]: The primary goal of assigning estimates is to make the work visible on the board and justify time allocation (e.g., "we spent 2 days on integration KT implementation"). The team has no formal planned integration work currently; these stories are driven by technical improvement and KT goals.

---

## Key Takeaways

1. **The root problem** is that connector `install`/`uninstall` functions are God functions — they mix CRM calls, DB writes, and DB deletes in a single nested call chain, making error handling (especially the `ignore_external_system_errors` flag) very painful to propagate.

2. **The fix** is to move orchestration responsibility up to the Integration Manager and break connector interfaces into single-responsibility functions (`registerWebhooks`, `storeCredentials`, `unregisterWebhooks`, `removeCredentials`, `createFolder`, `bootstrapAttributes`, `bootstrapEvents`).

3. **The `ignore_external_system_errors` flag** exists for two real scenarios: (a) operations bus account deletes, and (b) support-initiated force-uninstalls when customers have accidentally invalidated their CRM credentials. Both use the same code path.

4. **Changing the connector interface is a multiplier**: any new function added to the interface must be implemented in all connectors. Plan for this overhead.

5. **Start with FSC Enterprise** as the reference implementation, then extend to Dynamics, Lime, and Generic connectors.

6. **Webhook unregistration failures** have caused real production issues (orphaned DB data). Bootstrap failures have not. Prioritize webhook-related stories.

7. **A Postman collection** (`integration flow`) exists with requests for triggering force-uninstall and other direct Integration Manager calls. Erik to share this with the team.

---

## Unresolved Questions and Action Items

- [ ] **Erik** to polish the groomed stories in Jira and add any remaining technical refinement stories to the epic.
- [ ] **Erik** to share the `integration flow` Postman collection with the team.
- [ ] **Team** to align with product on prioritization before starting implementation, as integration work is currently competing with product-driven priorities.
- [ ] **Implementation sequence** for connector rollout (after FSC Enterprise) not yet decided — Dynamics, Lime, Generic Connector order TBD.
- [ ] ⚠️ **Ambiguity:** The exact names of the Jira epic and stories were being created live during the session and some ticket numbers were mentioned (e.g., "8401") but not definitively confirmed. Verify current state in Jira.