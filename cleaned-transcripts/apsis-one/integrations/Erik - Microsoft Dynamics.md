---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 connector, Azure AD / Entra tenant management, OAuth application registration, Dynamics environment setup, Plugin/solution installation, Integration architecture, On-premise support stance]
speakers: ["Erik Andersson (Integration domain expert)", "Lukasz Grabowski", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [Microsoft Dynamics 365 CRM (legacy connector), Dynamics by Sideshop (generic connector), Azure Entra (Azure AD), Admin Power Platform, AWS Secrets Manager (AVS), Integration Manager, One API, Apsis One front end, Dynamics plugin/solution, Power Apps, Dataverse]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors"]
---

## Session Overview

This KT session covers the legacy Microsoft Dynamics 365 CRM connector — the first Dynamics connector Apsis built in-house — as distinct from the newer "Dynamics by Sideshop" connector which implements the generic connector interface. Erik walks through the two Microsoft organizational tenants Apsis manages, the OAuth application registrations critical to all customer integrations, how to create a fresh Dynamics environment for development/testing, and the purpose and installation process for the Dynamics plugin (solution). The session ends before the plugin installation completes; a follow-up session was agreed.

---

## Legacy vs. New Dynamics Connector: Strategic Context

Two connectors are visible in the Apsis One UI:

1. **Microsoft Dynamics 365 CRM** — the legacy connector, built in-house by Apsis. This is the connector discussed in this session.
2. **Dynamics by Sideshop** — the new connector, managed by Sideshop, which implements the **generic connector** interface.

> "The recommendation is for all new customers to go down this route [Sideshop/generic] because this is more future proof for new features. Everything we add for the generic connector, there is a strong possibility that Sideshop will add to their connector, whereas we will not add any new functionality to the old connectors."

This has a direct operational implication: migrating existing customers from the legacy connector to the new one is desirable because Apsis currently owns the administration of two Microsoft tenants on behalf of legacy connector customers. Every time someone leaves Apsis, their access to these tenants must be manually revoked. Moving customers to the new connector reduces this administrative burden.

---

## The Two Microsoft Organizational Tenants Apsis Manages

Two separate Microsoft organizations (tenants) are critical to the legacy Dynamics integration. Access is via [entra.microsoft.com](https://entra.microsoft.com) and [admin.powerplatform.microsoft.com](https://admin.powerplatform.microsoft.com).

### Tenant 1 — Apsis Lab (`apsisut.onmicrosoft.com`)
- Used for **development and test environments**.
- Created and set up by **CRM Consultana** (the external consulting firm that helped develop the solution).
- This is where new Dynamics instances/environments are created.
- Apsis has permission to create new instances and has a set of licenses here.

### Tenant 2 — Apsis International (`apsis.onmicrosoft.com`)
- Stores the **OAuth application registrations** (app registrations) and their client secrets.
- This is where the credentials that every customer's integration depends on are managed.
- Accessible at [entra.microsoft.com](https://entra.microsoft.com) → switch directory to "Apsis International AB."

### User Access Model
- **Do not create new users** in these tenants. Always use **"Invite external user"** and supply the person's `fc.com` email address.
- When an external user is invited with their FC/FSC corporate account, their access is automatically revoked if their corporate account is centrally disabled — no manual cleanup needed in the Microsoft tenant.
- If you instead create a new internal user, you introduce a credential that is disconnected from the FC identity lifecycle.
- For the development tenant (Apsis Lab), users must be added as **Members** (not Guests). Guests cannot modify anything.
- To approve the OAuth application authorization step during Dynamics installation (see below), the installing user must have the **Global Administrator** role in the tenant.

---

## OAuth Application Registrations — Critical Infrastructure

Located in **Entra (`apsis.onmicrosoft.com`) → App Registrations → All Applications**.

### What These Are
These are the Azure AD application registrations that Apsis uses to authenticate against customer Dynamics instances. During installation, the customer grants Apsis's application permission to access their organization. After that, Apsis uses the app's **client ID + secret** to generate access tokens via the OAuth flow, which are then used to call the customer's Dynamics instance.

### The Most Important Applications

| App Name | Purpose |
|---|---|
| `Apsis One EU prod` | **Production EU customers** — single most critical app |
| `Apsis One APAC prod` | Production APAC customers |
| `Apsis One beta` | Beta environments |
| `Apsis staging 2` | Staging environments (the old staging app is expired) |

> "This is the single most important application of them all because we use this one to access all customer environments. It might not be the best approach because essentially it's the comparison of having one API secret for every customer account, but that is how it was set up for us."

One application is shared across **all** customers in that environment tier. Every production EU customer's integration depends on the `Apsis One EU prod` app registration.

### Secret Management
- Secrets are stored in **AWS Secrets Manager (AVS)**.
- The **Integration Manager** loads the secret on boot and uses it to generate access tokens.
- Secrets are currently set to **never expire**.
- The **client ID** for each app is also referenced in the **Apsis One front-end** configuration (environment-specific config files, e.g., a `DYNAMICS_CLIENT_ID` variable). Erik confirmed this is one of the few integration config values handled by the front end rather than the back end.

### What Happens If the Secret Is Lost or Needs Rotation
- If only the **secret value** needs to be regenerated (same application): create a new secret in Entra, update it in AVS. **Customers do not need to reinstall** — the secret is only used by Apsis when talking to Microsoft, not stored or used by customers directly.
- If a completely **new application registration** must be created (new client ID): every customer would need to re-approve the new application, effectively requiring a full reinstall of the integration for all affected customers. This is a catastrophic scenario.

### Real Incident: Customer Revoking Access
> "We just had a problem with a customer where they, on installation, had approved us to log in on their instance with that application, and then later someone in their IT said 'what is this, I'll disable it.' And then we lost complete access to their environment until they re-enabled it."

This is a known risk: customer IT administrators can revoke Apsis's application access at any time without Apsis being notified in advance.

---

## Creating a Development Dynamics Environment

Done via **Admin Power Platform** ([admin.powerplatform.microsoft.com](https://admin.powerplatform.microsoft.com)), logged in under the **Apsis Lab** tenant.

> Note: You must be logged in under the Apsis Lab tenant (`apsisut.onmicrosoft.com`), **not** Apsis International. Environments are managed in Apsis Lab; OAuth apps are managed in Apsis International.

### Steps to Create a New Environment
1. Navigate to **Manage → Environments** in the left menu.
2. Click **New** to create an environment.
3. Set a name (e.g., "Integration Development").
4. Set region to **Europe**.
5. Enable **Dataverse** data store (this is the database for the instance).
6. Set **Security Group** to **Open Access** (authentication is still required; this just means no additional group restriction).
7. **Enable Dynamics 365 apps** — this is critical. It enables:
   - Marketing lists
   - Lead handling inside Dynamics
   - Without this, only regular contacts are available; email list syncing will not work.
8. Press **Save**. The environment will spend a few minutes in "Preparing" state.

Once ready, the environment gets a URL like:

```
integrations-development.crm4.dynamics.com
```

### When Would You Need to Do This?
This is not a routine task. It is only needed if:
- You want a completely clean slate to test plugin installation/uninstallation.
- The existing test environment is in a bad state.
- Since no new development is happening on the legacy plugin, this should be exceedingly rare.

### On-Premise Dynamics
Apsis does **not** support fully on-premise Dynamics deployments.
> "If the customer has a completely locked-down on-premise [setup] which cannot talk with the internet, we don't support that. The official standpoint from Apsis is we don't support on-premise Dynamics."

Reason: the integration relies on Azure-hosted webhooks, the OAuth login flow through Azure, and other Azure resources that require internet connectivity.

---

## The Dynamics Plugin (Solution) — Architecture and Installation

### What the Plugin Is
The plugin is a **Dynamics Solution file** (`.zip`) provided to customers by the Apsis sales team before installation begins. It was developed by **CRM Consultana** according to Apsis's specifications.

There are two plugins, with the main one being `Apsis [something] managed`.

### What the Plugin Does
Installing the plugin into the customer's Dynamics instance does the following:

1. **Creates a Security Role** inside the customer's Dynamics instance. This role grants all permissions Apsis needs:
   - Read, update, and retrieve contacts
   - Operate at organizational (not just individual user) level

2. **Creates custom database tables** used for background operations, most critically to support the real-time sync flow.

3. **Implements a webhook system** inside Dynamics. Dynamics does not natively support outbound webhooks, so the plugin implements this:
   - It listens to events inside Dynamics.
   - If an event is for a **Contact**, it calls the **One API** with the profile data.
   - The call uses a **One API client ID and secret** that are generated during the installation process, sent to the plugin, and stored in one of the custom database tables.

### What the Installation Process Does (in sequence)
1. Customer installs the solution file via **Power Apps → Import Solution** (accessible from the Dynamics instance's **Advanced Settings** → **Solutions** menu → redirects to Make Power Apps).
2. During Apsis-side setup, an **Application User** is created inside the customer's Dynamics instance.
3. Apsis's OAuth application (e.g., `Apsis One EU prod`) is connected to this Application User.
4. The Security Role created by the plugin is attached to this Application User.
5. A **One API client ID and secret** are generated and stored in the plugin's custom table for use by the webhook background process.

Result: Apsis has an Application User in the customer's tenant, can generate access tokens for it via the Azure OAuth app, and that user has the security role granting all necessary permissions. The plugin's background process can also push real-time contact changes back to Apsis using the stored One API credentials.

### Consequence of Missing Plugin
> "If you don't have this plugin, nothing will work. You will fail to install, and even if you had managed to install, we would not have permission to read any contacts and no data would be sent to Apsis — it would be completely useless."

### Where to Find the Solution File
Erik noted he would show the repository location after the break (not covered in this session).

---

## Access and Permissions — Practical Notes for the Team

- **Lukasz** was unable to access the `apsis.onmicrosoft.com` Entra tenant during the session — likely a pending invitation acceptance or admin rights issue. To be resolved in a follow-up.
- **Tomasz and Michal** successfully gained access after switching directories and completing MFA setup for the new tenant.
- All three were added as **Global Administrators** during the session to ensure they can approve OAuth application authorization during test installations.
- The **Admin Power Platform** page may only show meaningful environment controls if the user is an admin — non-admin members may see a restricted view.

---

## Key Takeaways

1. **The legacy Dynamics connector is in maintenance mode** — no new features will be added. All new customers should use the generic connector (Dynamics by Sideshop).

2. **Two Microsoft tenants matter**: `apsisut.onmicrosoft.com` (dev/test environments) and `apsis.onmicrosoft.com` (OAuth app registrations). Both must be accessible to the integration team.

3. **The `Apsis One EU prod` OAuth app registration is the single most critical credential** in the entire legacy Dynamics integration. Loss of this app or its secret would require every EU production customer to reinstall their integration.

4. **Secrets are in AWS Secrets Manager; client IDs are in the Apsis One front-end config**. Secret rotation only requires updating AVS and the Integration Manager — customers are unaffected. A new app registration would require customer reinstalls.

5. **The Dynamics plugin is a prerequisite for everything**: permissions, real-time sync webhooks, and custom data tables are all provided by it. Without it, the integration cannot function at all.

6. **Global Administrator role** is required in the customer's tenant for the one-time step of approving Apsis's OAuth application. This is historically the biggest friction point with enterprise customers' IT departments.

7. **On-premise Dynamics is not supported** due to dependency on Azure-hosted OAuth and webhook infrastructure.

8. **Always invite external users** (using FC corporate accounts) to Apsis's Microsoft tenants — never create new internal users — to ensure access is automatically revoked when someone leaves the company.

---

## Unresolved Questions / Action Items

- [ ] **Lukasz's tenant access**: Lukasz could not switch to `apsis.onmicrosoft.com` in Entra or access Admin Power Platform under Apsis Lab. Erik to investigate and fix in a separate call.
- [ ] **App registration cleanup**: Erik to audit the `apsis.onmicrosoft.com` app registrations, remove expired/unknown entries, and produce a list of which applications are actively in use.
- [ ] **Stale global admins cleanup**: Several Global Administrator accounts in the Apsis Lab tenant belong to people who no longer work at Apsis (e.g., Zenita Pelko). To be removed.
- [ ] **Follow-up session**: This session ended before covering where to find the solution file in the repository, the full installation walkthrough (the plugin import was kicked off but not completed), and remaining topics. A follow-up session was agreed — Lukasz to schedule.
- [ ] **Plugin repository location**: Erik to show where the solution file lives in source control (deferred to next session).