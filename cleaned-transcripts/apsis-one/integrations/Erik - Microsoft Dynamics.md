---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 CRM legacy connector, OAuth2 application registration, Azure Entra ID tenant management, Dynamics environment provisioning, Dynamics solution/plugin installation, real-time sync via webhook emulation, integration architecture overview]
speakers: ["Erik Andersson (Integration Developer/Owner)", "Lukasz Grabowski (Handover Recipient)", "Michal Rosikiewicz (Handover Recipient)", "Tomasz Kowalski (Handover Recipient)"]
key_components: [Microsoft Dynamics 365 CRM legacy connector, Dynamics by Sideshop (generic connector), Azure Entra ID, Admin Power Platform, Apsis One Integration Manager, AWS Secrets Manager, OAuth2 app registrations, Dynamics Solution/Plugin, Power Apps]
session_type: knowledge-transfer
---

## Session Overview

This session is a knowledge transfer from Erik Andersson covering the **legacy Microsoft Dynamics 365 CRM connector** in the Apsis One integration platform. Erik walks through the two Azure tenants critical to the integration, the OAuth2 application registrations used for all customer environments, how to provision a new Dynamics test environment, and the architecture of the Dynamics plugin/solution that must be installed in the customer's instance. The session also covers the real-time sync mechanism, permission model, and the rationale for migrating customers to the newer generic connector. The session was paused mid-way through the plugin installation step and is intended to continue in a follow-up.

---

## Legacy vs. New Dynamics Connector — Architecture Decision

There are two Dynamics connectors in the Apsis One platform:

1. **Microsoft Dynamics 365 CRM** — the legacy connector, subject of this session.
2. **Dynamics by Sideshop** — the new connector, which implements the **generic connector** interface and is managed by Sideshop.

> "The recommendation is for all new customers to go down this route [Sideshop/generic connector] because this is more future proof for new features. Everything we add for the generic connector, there is a strong possibility that Sideshop will add this to their connector, whereas we will not add any new functionality to the old connectors."

**Key implication:** No new feature development is planned for the legacy connector. This also reduces the frequency with which anyone needs to touch the legacy infrastructure.

---

## Azure Tenant Architecture — Two Critical Organizations

There are two distinct Microsoft organizations (tenants) that must be understood:

### Tenant 1: `apsisut.onmicrosoft.com` (Development Tenant — "Apsis Lab")
- Used for **creating and managing Dynamics environments/instances**.
- Managed via [admin.powerplatform.microsoft.com](https://admin.powerplatform.microsoft.com).
- This account was created by **CRM Consultana**, the external firm that helped develop the solution.
- Apsis has permission to create new instances and has a set of licenses here.
- Referred to internally as "**Apsis Lab**" or "Apsis Development."

### Tenant 2: `apsis.onmicrosoft.com` (Credentials Tenant)
- Used for **storing OAuth2 application registrations and secrets**.
- Managed via [entra.microsoft.com](https://entra.microsoft.com) (formerly Azure Active Directory).
- This is where the OAuth app client IDs and secrets live for all customer-facing environments.

**Why this split matters:** If you lose access to the credentials tenant and cannot retrieve secrets from AWS Secrets Manager, every customer using the legacy Dynamics connector would need to reinstall their integration. This is a critical single point of failure.

### Accessing the Tenants
Team members use their `@fc.com` (FSC) corporate accounts as **external/guest users** — they are not given standalone accounts within these organizations. This is intentional:

> "If you invite an external user and you invite their FC user, then when that user is centrally disabled at FC, the access to your tenants is also removed. So there are only advantages of doing that."

When inviting users in Entra, always choose **"Invite external user"** — not "Create new user." Set the role to **Member** (not Guest) so the user can modify things based on their assigned roles.

To switch between tenants after being invited: go to [entra.microsoft.com](https://entra.microsoft.com), click the account icon top-right, and use **Switch Directory**.

> ⚠️ **Admin note:** There were several former employees still present as global admins in these tenants at the time of this session. Erik flagged this as a cleanup item. The fact that FSC accounts are used mitigates the security risk (disabled FSC accounts lose access), but it looks bad in an audit.

---

## OAuth2 Application Registrations — Critical Production Secrets

Located in: [entra.microsoft.com](https://entra.microsoft.com) → **App registrations** → **All applications**, under the `apsis.onmicrosoft.com` tenant.

### Key applications in use

| App Name | Purpose |
|---|---|
| `Apsis One EU Prod` | **Primary production app — used for all EU customer environments** |
| `Apsis One APAC Prod` | Used for all APAC customer environments |
| `Apsis One Staging 2` | Current staging app (the old staging app is expired) |
| `Apsis One Beta` | Beta environments |

> ⚠️ **This is a single shared application per environment tier.** One OAuth app is used to access ALL customer Dynamics instances in that environment. This is architecturally equivalent to having one API secret for every customer account.

> "If this one were to disappear for whatever reason, then that's a very annoying situation, to say the least."

There are many additional expired/legacy app registrations in the list. Erik noted he would clean these up and document which are actively in use.

### How the OAuth secret is used

The flow is:
1. Secret is stored in **AWS Secrets Manager**.
2. The **Integration Manager** loads the secret on boot.
3. The Integration Manager uses the secret to make a request to Azure to generate an **access token**.
4. That access token is used to communicate with the customer's Dynamics instance.

> "The secret is for us. The customer doesn't store the secret or anything. The secret is used by us when we authenticate — we do the OAuth flow to the Microsoft service."

### Rotating the secret vs. rotating the application

- **Rotating the secret only:** Apsis updates the secret in AWS Secrets Manager and redeploys/restarts the Integration Manager. **Customers do not need to do anything.**
- **Creating a completely new application (new App ID):** Every customer would need to re-approve the new application for access to their organization — i.e., a full reinstall. **This must be avoided at all costs.**

To create a new secret: App registration → **Certificates & secrets** → create new secret. The current secrets are set to never expire.

### Real-world incident: Customer disabled the app access

> "We just had a problem with a customer where they had approved us to log in to their instance with that application on installation, and then later someone in their IT said 'what is this?' and disabled it. We lost complete access to their environment until they re-enabled it."

This is a known operational risk — customer IT departments may revoke access unilaterally.

### Where the App Client ID is configured in the Apsis codebase

The OAuth App Client ID is referenced in the **front-end repository** global configuration, not in the back end. Erik showed a search result confirming the Client ID for each environment is set in the front-end config under something like `DYNAMICS_CLIENT_ID`. Example:

- Staging uses an ID starting with `B66605...`
- Prod EU has its own corresponding ID

---

## Provisioning a New Dynamics Test Environment

Done via: [admin.powerplatform.microsoft.com](https://admin.powerplatform.microsoft.com) → **Manage** → **Environments** → **New**

You must be logged in under the **Apsis Lab** (`apsisut.onmicrosoft.com`) tenant to create environments.

### Environment creation settings

| Setting | Value / Notes |
|---|---|
| Name | e.g., `Integration Development` |
| Region | Europe |
| Add Dataverse data store | **Yes** — this is the database for the instance |
| Language | English |
| Security group | **Open Access** — authentication is still required; this just controls who can be added |
| Enable Dynamics 365 apps | **Yes — critical** |

### Why "Enable Dynamics 365 apps" is required

> "This enables more functionality and more precisely it enables Marketing lists and lead handling inside Dynamics. If you don't do this, then you can only handle your regular contacts. The integration supports marketing lists inside Dynamics — the email list syncing. If you don't enable this then we can't access them."

Environment creation takes approximately 1–2 minutes. Once ready, the environment URL follows the pattern:

```
<environment-name>.crm4.dynamics.com
```

The `crm4` segment indicates the European region.

### When you would need to do this

This is not a routine task. You would provision a new environment if:
- You need a clean slate to test installation/uninstallation of the plugin.
- The existing QA environment becomes corrupted or needs replacement.

> ⚠️ **Do not touch the QA environment** (`Integrations QA`) — it is used by QAs for manual testing and automated test suites.

### On-premises Dynamics — Not Supported

> "We don't support on-premise Dynamics. We use resources inside of Azure — web hooks in Azure, the OAuth login flow in Azure. If the customer has a completely locked-down on-premise system that cannot talk with the Internet, we don't support that. The official standpoint from Apsis is we don't support on-premise Dynamics."

---

## The Dynamics Solution/Plugin — Architecture and Installation

### What is a "Solution" in Dynamics

In Dynamics terminology, a plugin/extension is called a **Solution**. It is a deployable package installed into a Dynamics environment via the Power Apps interface.

**Path to install:** Dynamics instance → cog wheel (⚙️) → **Advanced Settings** → **Solutions** → redirects to **Power Apps** → **Import Solution**

> Note: The UI in Dynamics updates frequently without notice. Erik observed the interface had already changed from what he expected even on a freshly created instance.

### Where to find the solution file

The solution file (`.zip`) was provided by **CRM Consultana** (the external development partner) and is developed to Apsis specifications. Erik noted he would show the file location in a follow-up session.

There are **two solution packages**:

1. **Main plugin** (`apsis_<something>_managed.zip` or similar name) — this is the critical one.
2. A second supplementary plugin (details to be covered in follow-up).

### What the main plugin does

The plugin does several things that are **prerequisites for the integration to function at all**:

1. **Creates a Security Role** inside the customer's Dynamics instance with all required permissions:
   - Read contacts
   - Update contacts
   - Get all contacts
   - All operations at **organizational level** (not just per-user)

2. **Creates custom database tables** used for internal background processes.

3. **Implements a webhook emulation system** — Dynamics does not natively support outbound webhooks. The plugin creates a background process that:
   - Listens to events inside Dynamics
   - Checks if the event is for a Contact
   - If so, contacts the Apsis One API with the profile data (real-time sync)

4. As part of the **installation process** (separate from the plugin install itself): a One API client ID and secret are generated, sent to Dynamics, and stored in one of the plugin's custom tables. The background process uses this secret when sending real-time notifications to Apsis.

### Installation process summary (requires the plugin to be installed first)

1. Customer installs the plugin into their Dynamics environment.
2. Apsis creates an **Application User** inside the customer's Dynamics instance.
3. Apsis connects the OAuth app (from `apsis.onmicrosoft.com` app registrations) to this Application User.
4. Apsis attaches the **Security Role** (created by the plugin) to this Application User.
5. Result: Apsis can generate an access token via our Azure app and authenticate as this Application User with the required permissions.

### Why the plugin is non-negotiable

> "If you don't have this plugin, nothing will work. You will fail to install, and even if you had managed to install, we would not have permission to read any contacts and no data would be sent to Apsis. It would be completely useless."

### Global administrator requirement for first-time OAuth approval

> "Only global admins can approve the authorization of the app connecting to your organization. If you have done this once, then you don't need to be an admin to install Dynamics, but the first time this happens you need to have a global administrator approving it. This tends to be the biggest challenge for customers — getting their IT people to want to or get time to do this for them."

---

## Operational Concern: Tenant Administration Overhead

The legacy connector architecture has an operational burden: Apsis effectively **owns the administration of two Microsoft tenants**. This means:
- When employees leave Apsis, they must be manually removed from these tenants.
- While disabled FSC accounts lose access automatically (mitigating the security risk), active auditing would flag the stale entries.

> "This is the second reason why it is in your interest that we migrate old customers to the new Dynamics — right now we own the administration of these two tenants."

The new Sideshop/generic connector does not have this burden.

---

## Key Takeaways

1. **All new customers should be directed to the Sideshop/generic connector.** The legacy connector receives no new feature development and carries significant administrative overhead.
2. **Two Azure tenants are critical:** `apsisut.onmicrosoft.com` (environments) and `apsis.onmicrosoft.com` (OAuth credentials). Losing access to either is a serious incident.
3. **The OAuth app `Apsis One EU Prod` is the single most critical credential artifact** for EU production — it is shared across all customer environments. It must never be deleted, and its secret must always be accessible in AWS Secrets Manager.
4. **Rotating the secret ≠ creating a new app.** Only creating a new app requires customer re-installation; secret rotation is transparent to customers.
5. **The Dynamics plugin/solution is a hard prerequisite** for any functionality — it sets up the security role, custom tables, and the webhook emulation background process.
6. **On-premise Dynamics is not supported** due to dependency on Azure-hosted webhooks and OAuth flows.
7. **"Enable Dynamics 365 Apps"** must be checked when creating test environments, otherwise marketing list functionality is unavailable.
8. **Always invite team members as external users** (not new users) to the Azure tenants, using their corporate FC accounts, to ensure access is revoked automatically on departure.

---

## Unresolved Questions and Action Items

- [ ] **Erik:** Lukasz was unable to access the `apsis.onmicrosoft.com` tenant via directory switching — to be resolved in a separate individual session.
- [ ] **Erik:** Clean up expired and unknown app registrations in `apsis.onmicrosoft.com` → App registrations, and document the definitive list of active apps.
- [ ] **Erik:** Remove former employees from global admin roles in both tenants (Zenita Pelko identified; others flagged).
- [ ] **Erik/Team:** Grant Lukasz, Michal, and Tomasz admin access to Admin Power Platform (environment management).
- [ ] **Follow-up session needed:** Session ended during plugin import. Remaining topics to cover:
  - Location of the solution `.zip` files in the repository
  - Full walkthrough of the plugin installation and the post-install configuration steps
  - Full customer installation flow end-to-end
  - One API client ID/secret generation and storage in Dynamics custom tables